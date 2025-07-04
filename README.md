# Despliegue de WordPress y MySQL en Kubernetes

Este repositorio contiene los manifiestos YAML necesarios para desplegar una instancia de WordPress con una base de datos MySQL en un clúster Kubernetes. La configuración utiliza volúmenes persistentes basados en `hostPath` para la persistencia de datos y expone WordPress mediante un servicio de tipo `NodePort`.

## Contenido del Manifiesto `wordpress-mysql.yaml`

El manifiesto `wordpress-mysql.yaml` incluye los siguientes recursos de Kubernetes:

1.  **Namespace `wordpress`**: Un espacio de nombres dedicado para organizar todos los recursos de la aplicación.
2.  **PersistentVolumeClaim (PVC) para MySQL (`mysql-pv-claim`)**: Solicita un volumen persistente de 5GB para almacenar los datos de la base de datos MySQL. Se enlaza a un `PersistentVolume` con `storageClassName: manual`.
3.  **PersistentVolumeClaim (PVC) para WordPress (`wordpress-pv-claim`)**: Solicita un volumen persistente de 5GB para almacenar los archivos del sitio web de WordPress. Se enlaza a un `PersistentVolume` con `storageClassName: manual`.
4.  **Deployment de MySQL (`mysql`)**: Define el despliegue del pod de MySQL (versión 5.7). Configura las variables de entorno para la contraseña de `root`, el nombre de la base de datos (`wordpressdb`), el usuario (`wpuser`) y la contraseña (`Yanky.2012`). Monta el PVC `mysql-pv-claim` en `/var/lib/mysql`.
5.  **Service de MySQL (`mysql`)**: Un servicio interno de tipo `ClusterIP` (headless) que permite al pod de WordPress conectarse a la base de datos MySQL usando el nombre de host `mysql` en el puerto 3306.
6.  **Deployment de WordPress (`wordpress`)**: Define el despliegue del pod de WordPress (última versión). Configura las variables de entorno para la conexión a la base de datos (host, nombre, usuario, contraseña). Monta el PVC `wordpress-pv-claim` en `/var/www/html`.
7.  **Service de WordPress (`wordpress`)**: Un servicio de tipo `NodePort` que expone la aplicación WordPress al exterior del clúster en el puerto 80 del contenedor, mapeado al puerto `30080` en los nodos del clúster.

## Prerrequisitos

Antes de desplegar esta aplicación, asegúrate de cumplir con los siguientes puntos en tu clúster Kubernetes:

* Un clúster Kubernetes funcional con al menos un nodo Master y un nodo Worker en estado `Ready`.
* **Persistent Volumes Manuales:** Dado que este manifiesto asume un `storageClassName: manual`, debes haber creado previamente los `PersistentVolume` correspondientes. Esto implica:
    * Tener un archivo `pv-manual.yaml` (no incluido en este manifiesto, pero necesario) que defina `mysql-pv` y `wordpress-pv` con `storageClassName: manual` y `hostPath` (ej., `/mnt/data/mysql` y `/mnt/data/wordpress`).
    * Haber creado los directorios `/mnt/data/mysql` y `/mnt/data/wordpress` en **todos los nodos** de tu clúster (Master y Worker) que puedan alojar pods, y haberles asignado permisos de escritura (ej., `sudo chmod 777 /mnt/data/mysql /mnt/data/wordpress`).
* **`kubectl` configurado** en tu máquina de control (Master) para interactuar con el clúster.

## Instrucciones de Despliegue

Sigue estos pasos desde el nodo Master de tu clúster Kubernetes:

1.  **Clonar este repositorio:**
    ```bash
    git clone [https://github.com/Franco-Canales/worpress.git](https://github.com/Franco-Canales/worpress.git)
    cd worpress
    ```

2.  **Asegurar los directorios de almacenamiento en los nodos:**
    Verifica que los directorios `/mnt/data/mysql` y `/mnt/data/wordpress` existen y tienen los permisos correctos (`chmod 777`) en **todos tus nodos** del clúster (Master y Worker).

3.  **Configurar Persistent Volumes (si usas `hostPath`):**
    Si no tienes un StorageClass dinámico, asegúrate de que tu `pv-manual.yaml` (el archivo que define los PVs estáticos) está en el directorio `manifests/`. Si no lo tienes, créalo con el contenido proporcionado en la documentación del despliegue del clúster.
    ```bash
    kubectl apply -f manifests/pv-manual.yaml
    ```

4.  **Configurar Contraseñas (¡CRÍTICO!):**
    Edita el archivo `manifests/wordpress-mysql.yaml` para establecer tus propias contraseñas seguras.
    ```bash
    nano manifests/wordpress-mysql.yaml
    ```
    * Modifica las líneas:
        * `value: "Yanky.2012"` para `MYSQL_ROOT_PASSWORD` (cambia a tu contraseña de root de MySQL).
        * `value: "Yanky.2012"` para `MYSQL_PASSWORD` (cambia a tu contraseña para el usuario `wpuser`).
        * `value: "Yanky.2012"` para `WORDPRESS_DB_PASSWORD` (cambia a tu contraseña para WordPress).
    * **IMPORTANTE:** Las contraseñas para `MYSQL_PASSWORD` y `WORDPRESS_DB_PASSWORD` **DEBEN COINCIDIR EXACTAMENTE**.
    * Guarda los cambios y sal del editor.

5.  **Aplicar el Despliegue de WordPress y MySQL:**
    ```bash
    kubectl apply -f manifests/wordpress-mysql.yaml
    ```

6.  **Verificar el Estado de los Recursos:**
    Monitorea el estado de los recursos para asegurarte de que todo se despliega correctamente. Puede tomar unos minutos para que las imágenes se descarguen y los pods se inicien.
    ```bash
    kubectl get namespace wordpress
    kubectl get pvc -n wordpress
    kubectl get deployments -n wordpress
    kubectl get services -n wordpress
    kubectl get pods -n wordpress -o wide --watch # Presiona Ctrl+C cuando los pods estén Running
    ```
    Asegúrate de que los PVCs estén `Bound` y los pods estén `Running` (ej., `1/1 READY`).

7.  **Acceder a WordPress desde tu Navegador:**
    Una vez que el pod de WordPress esté `Running`, abre tu navegador y visita:
    ```
    http://<IP_DE_TU_NODO>:30080
    ```
    Reemplaza `<IP_DE_TU_NODO>` con la dirección IP de cualquiera de tus nodos de Kubernetes (Master o Worker, ej., `192.168.150.141` o `192.168.150.137`). El puerto `30080` es el `NodePort` configurado para el servicio de WordPress.

    Sigue las instrucciones en pantalla de WordPress para completar la instalación, usando los detalles de la base de datos que configuraste en el YAML (`wordpressdb`, `wpuser`, y tu contraseña).

## Notas Importantes

* **HostPath para Volúmenes:** El uso de `hostPath` es adecuado para entornos de desarrollo y prueba. Para entornos de producción, se recomienda encarecidamente utilizar un aprovisionador de almacenamiento dinámico (CSI driver) con un sistema de almacenamiento de red robusto (NFS, Ceph, AWS EBS, Google Persistent Disk, etc.) para garantizar la alta disponibilidad y la resiliencia de los datos.
* **Gestión de Contraseñas:** En un entorno de producción real, las contraseñas no deben estar directamente en los manifiestos YAML. En su lugar, se deben usar [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) para almacenar información sensible de forma segura y referenciarlas en los Deployments.
* **Versiones de Imágenes:** Las imágenes `mysql:5.7` y `wordpress:latest` se utilizan en este manifiesto. Para entornos de producción, es una buena práctica especificar versiones de imagen más precisas y estables (ej., `wordpress:6.5.4-php8.2-apache`).
