# FRC - Fundamentos de redes y computación en la nube

Este laboratorio se centra en la práctica de conceptos de redes y computación en la nube mediante el uso de VPC y EC2 en AWS, para luego pasar a la publicación de contenido estático en S3.

## Resultados esperados

Al completar el laboratorio podrá:

1. Reconocer VPC, subnet, route table e Internet Gateway en una red propia
2. Lanzar una instancia EC2 dentro de una subnet pública
3. Conectarse desde el navegador mediante EC2 Instance Connect
4. Publicar un servidor HTTP y asociarle una Elastic IP
5. Crear un bucket S3
6. Ajustar los permisos de S3 para poder publicar un sitio estático
7. Publicar un sitio estático mediante una bucket policy
8. Eliminar todos los recursos utilizados

## Preparación previa

Antes del laboratorio:

- Active su cuenta de AWS Academy
- Confirme que tiene acceso al curso **AWS Academy Learner Lab**

## Actividad

### 1. Iniciar AWS Academy Learner Lab

1. Ingrese a AWS Academy y abra **AWS Academy Learner Lab**
2. Seleccione **Modules** y luego **Launch AWS Academy Learner Lab**
3. Si aparecen, acepte los términos de uso
4. Haga click en **Start Lab** y espere hasta que el indicador al lado de **AWS** cambie a color verde
5. Haga click sobre **AWS** para acceder a la consola de AWS

> La consola se abrirá en una nueva pestaña. En la esquina superior derecha podrá ver el nombre de la cuenta y la región. Confirme que la región sea **`us-east-1`** antes de continuar. Si no lo es, cambie la región en el menú desplegable.

### 2. Crear una VPC y recursos asociados

#### 2.1 Crear la VPC y recursos asociados

1. En el buscador superior busque y abra **VPC**
2. En **Your VPCs**, seleccione **Create VPC**
3. En **Resources to create**, seleccione **VPC and more**
4. Configure:
    | Campo | Valor |
    | --- | --- |
    | Name tag auto-generation | `tel351` |
    | IPv4 CIDR block | `10.10.0.0/16` |
    | IPv6 CIDR block | No IPv6 CIDR block |
    | Tenancy | Default |
    | Number of Availability Zones | 1 |
    | Number of public subnets | 1 |
    | Number of private subnets | 2 |
    | NAT gateways | None |
    | VPC endpoints | None |
    | Enable DNS hostnames | Enabled |
    | Enable DNS resolution | Enabled |
5. Seleccione **Create VPC**
6. Espere hasta que todas las operaciones indiquen **Success**
7. Seleccione **View VPC**

#### 2.2 Reconocer los recursos

En la pestaña **Resource map**, identifique:

- La VPC `tel351-vpc`
- La subnet pública
- Las subnets privadas
- Las route table asociadas a las subnets

Revise las configuraciones de cada una de las route tables e infiera qué permite cada una de las reglas configuradas.

### 3. Publicar un servidor HTTP en EC2

#### 3.1 Lanzar la instancia

1. Busque y abra **EC2**
2. En **Instances**, seleccione **Launch instances**
3. Configure:
    | Campo | Valor |
    | --- | --- |
    | Name | `tel351-web` |
    | Application and OS Images | Ubuntu 26.04, arquitectura `64-bit (x86)` |
    | Instance type | `t3.small` |
    | Key pair | Proceed without a key pair |
    | VPC | `tel351-vpc` |
    | Subnet | La subnet pública creada anteriormente |
    | Auto-assign public IP | Enable |
    | Security group | Create security group |
    | Security group name | `tel351-web-sg` |
4. En las Inbound Security Group Rules dejar como está
5. Mantenga el almacenamiento raíz propuesto (8GiB gp3)
6. Seleccione **Launch instance**, espere y luego **View all instances**
7. Seleccione la instancia `tel351-web` y confirme: 
   - Instance state: Running
   - Status check: 2/2 checks passed
   - Revise los diferentes paneles y qué información contienen

#### 3.2 Conectarse desde la consola

1. Seleccione la instancia `tel351-web`, esto habilitará el boton **Connect** en la parte superior
2. Seleccione **Connect**
3. Seleccione la opción **EC2 Instance Connect** (debería estar seleccionada por omisión)
4. Confirme el usuario `ubuntu`
5. Seleccione **Connect**

Esto abrirá un nuevo tab con un terminal conectado a la instancia.

Ahora podemos utilizar nuestra infraestructura. Lo primero es actualizar el sistema:

```bash
sudo apt update 
sudo apt upgrade -y
```

Luego instalamos el servidor HTTP Apache y lo habilitamos para que inicie automáticamente:

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
```

Creamos una página de prueba para confirmar que el servidor funciona:

```bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="es">
<head><meta charset="utf-8"><title>FRC — EC2</title></head>
<body>
  <h1>Servidor HTTP en Amazon EC2 de ###Nombre###</h1>
  <p>Esta respuesta proviene de una instancia dentro de mi VPC.</p>
</body>
</html>
EOF
```

Confirme el servicio:

```bash
systemctl is-active apache2
```

El resultado debe ser `active`.

#### 3.3 Probar y asociar una Elastic IP

1. En los detalles de la instancia copie **Public IPv4 address**
2. Abra `http://<public-ipv4>` en una pestaña nueva
3. El navegador debe fallar, esto se debe a que el Security Group no permite tráfico HTTP
4. Agregue una regla de entrada al Security Group `tel351-web-sg`:
    1. Seleccione la intancia y luego **Security → Security groups**
    2. Seleccione el Security Group `tel351-web-sg`
    3. En **Inbound rules**, seleccione **Edit inbound rules**
    4. Seleccione **Add rule**
    5. Configure:
        - **Type:** HTTP
        - **Protocol:** TCP (autoseleccionado)
        - **Port range:** 80 (autoseleccionado)
        - **Source:** 0.0.0.0/0
5. Con el cambio, confirme que aparece la página creada

Con lo anterior está validado que la instancia puede recibir tráfico HTTP. Ahora se asociará una Elastic IP para que la dirección pública no cambie.

1. En EC2 abra **Network & Security → Elastic IPs**
2. Seleccione **Allocate Elastic IP address**
3. Mantenga **Amazon's pool of IPv4 addresses** y **Network border group** y seleccione **Allocate**
4. Seleccione la nueva dirección y luego **Actions → Associate Elastic IP address**
5. En **Resource type**, seleccione **Instance** (debería estar seleccionado por omisión)
6. En **Instance**, seleccione `tel351-web`
7. Vuelva a las instancias y seleccione `tel351-web`, note que la **Public IPv4 address** cambió a la Elastic IP
8. Abra `http://<elastic-ip>` y confirme la misma respuesta

### 4. Publicar contenido selectivamente en S3

#### 4.1 Crear un bucket

1. Busque y abra **S3**
2. Dado que no hay ningún bucket, el menu lateral está colapsado. Expándalo y seleccione **General purpose buckets**
3. Seleccione **Create bucket** y use un nombre globalmente único, por ejemplo `tel351-<rol>`
4. Confirme la Región **US East (N. Virginia) `us-east-1`**
5. Mantenga **ACLs Disabled**
6. Mantenga **Block all public access** habilitado
7. Mantenga las demás opciones por omisión y cree el bucket
8. Cargue `index.html` y `private.txt` disponibles en este repositorio en la raíz del bucket (puede hacer drag & drop o usar **Upload**)

#### 4.2 Observar el estado privado inicial

1. Abra `index.html` y copie su **Object URL**. Analice como está formada
2. Pruebe la URL en un navegador
3. Repita con `private.txt`

Ambas solicitudes deben responder **Access Denied**. La existencia de una Object URL no concede acceso anónimo.

#### 4.3 Habilitar el website endpoint

1. Abra la pestaña **Properties** del bucket
2. En **Static website hosting**, seleccione **Edit**
3. Seleccione **Enable** y **Host a static website**
4. En **Index document**, escriba `index.html`
5. Guarde los cambios
6. Copie el nuevo **Bucket website endpoint** y compare con la Object URL de `index.html`
7. Abra el endpoint en una pestaña nueva

Debe continuar recibiendo **403 Forbidden**, porque todavía no se ha concedido acceso público.

#### 4.4 Publicar únicamente `index.html`

1. Vuelva a la ventana del bucket y abra **Permissions → Block public access (bucket settings) → Edit**
2. Deshabilite **Block all public access** (esto debería de-seleccionar todas las opciones) y seleccione **Save changes**
3. Confirme la advertencia y guarde
4. En **Bucket policy**, seleccione **Edit**
5. Reemplace `BUCKET_NAME` en la policy siguiente y guárdela:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "PublicReadOnlyForIndex",
          "Effect": "Allow",
          "Principal": "*",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::BUCKET_NAME/index.html"
        }
      ]
    }
    ```

Compruebe:

- El **Bucket website endpoint** muestra el sitio
- La Object URL de `index.html` también responde
- La Object URL de `private.txt` continúa respondiendo **Access Denied**

Las ACL siguen deshabilitadas. La diferencia se explica exclusivamente mediante la bucket policy.

#### 4.5 Compartir temporalmente el objeto privado

1. Vuelva al bucket en S3 y en **Objects**, seleccione `private.txt`
2. Abra **Object actions → Share with a presigned URL**
3. Seleccione una duración de 1 minuto
4. Seleccione **Create presigned URL**
5. Pegue la URL en una ventana privada

Debe poder leer el objeto sin modificar la bucket policy. La URL utiliza los permisos de la identidad que la generó y deja de ser válida después de su expiración. Luego de 1 minuto, la URL debe responder **Access Denied**.

### 5. Limpieza

Realice el procedimiento en este orden.

#### 5.1 S3

1. Vaya a la lista de todos los buckets y seleccione el bucket creado
2. Haga click en **Delete**. La operación debe fallar ya que hay objetos dentro del bucket, y solo se puede eliminar un bucket vacío
3. Para vaciar el bucket hay 2 opciones:
    - Ingresar al bucket, seleccionar todos los objetos y usar **Delete**
    - Usar **Empty bucket** en la vista general del bucket
4. Una vez vaciado el bucket, regrese a **General purpose buckets**
5. Seleccione el bucket y elija **Delete**. Confirme el nombre del bucket y seleccione **Confirm**. El bucket debe desaparecer de la lista.

#### 5.2 Elastic IP

1. En EC2 abra **Elastic IPs**
2. Seleccione la dirección
3. Use **Actions → Disassociate Elastic IP address**
4. Use **Actions → Release Elastic IP addresses**

#### 5.3 EC2 y VPC

1. En **Instances**, seleccione `tel351-web`
2. Use **Instance state → Terminate instance**
3. Espere hasta que aparezca `Terminated`
4. Abra VPC, seleccione **Your VPCs** y seleccione `tel351-vpc`
5. Use **Actions → Delete VPC**
6. Revise los recursos que AWS eliminará y confirme

#### 5.4 Verificación final

Antes de salir confirme:

- No existe la Elastic IP
- El bucket fue eliminado
- La instancia aparece terminada
- La VPC `tel351-vpc` fue eliminada

#### 5.5 Terminar laboratorio

1. Cierre la pestaña de la consola de AWS
2. En AWS Academy, vuelva a la pestaña desde donde inició el laboratorio, seleccione **End Lab** y confirme

> AWS Academy levanta sesiones temporales cada vez que se inicia el laboratorio. La sesión dura 4 horas, por lo que si se le olvida terminar el laboratorio, la sesión se cerrará automáticamente; no obstante, los recursos creados no se eliminarán y podrían generar costos. Por ello, es importante realizar la limpieza de recursos independientemente de si la sesión es terminada manualmente o no
