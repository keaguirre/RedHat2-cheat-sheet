# Repaso examen RHEL

# Unidad 1
## SSH

- Server

        dnf install -y openssh-server
        vim /etc/ssh/sshd_config
        -> PermitRootLogin no
- Cliente

        ssh-keygen -t rsa -b 2048
        ls .ssh/
        cat .ssh/id_rsa #Clave privada
        cat .ssh/id_rsa.pub #Clave publica
        ssh-copy-id -i [ubicacion-clave-publica] [user-destino]@[ip-destino]
        ssh [user-destino]@[ip-destino] o ssh -i [ubicacion-clave-publica] [user-destino]@[ip-destino]

## Rsyslog
    vim /etc/rsyslog.conf
        -> [facility].[priority] [ruta-archivo-log]
        ej: local2.*    /var/log/local2.log #Esta linea dice que para este facility cualquier priority debera ir a ese archivo de log
    logger -p [facility].[priority] "mensaje de log"
    cat [archivo de log]

## Logrotate
    Configuracion de directorio o archivo de rotado:
        vim /etc/logrotate.conf
            agregar el siguiente bloque:
            [ruta-log-file]{
                [options]
            }
            ej: /var/log/secure{
                size 10M
                rotate 5
                compress
            }
            ej2:/var/log/secure{
                    rotate 5 #rota 5 veces antes de empezar a eliminar
                    compress # el resultado sera un archivo comprimido
                    truncate # crea una copia y vacia el archivo original
                    daily #diario
                    size #tamano
                    notifempty # si esta vacio no se genera la rotacion
                }
    Comprobacion de rotado:

    sudo dd if=/dev/zero of=/var/log/secure bs=1M count=11 #Llena el log a 11 megas
    logrotate -f /etc/logrotate.conf #fuerza el rotado
	ls /var/log/secure | grep secure #para verificar que no hay mas de 5 archivos comprimidos que se va borrando el 5to mas longevo para generar el nuevo mas actual
	ls -lh -> lista con el peso

## SCP
    sudo dd if=/dev/zero of=/var/log/secure bs=1M count=5120 #en caso de que necesite un archivo de 5gb para copiar
    [ruta-archivo-local] [usuariossh-server]@[ip-sersver]:[ruta/destino-server]
    ej: sudo scp /home/keaguirre/scp-file keaguirre-ssh@192.168.20.144:/home/keaguirre-ssh/scp-file

## Busqueda de archivos
- find:

        ls /var/log | grep messages
        find /var/log -name "messages"
        find /home/keaguirre-ssh/ -size +100M

- locate:

	    ls -lh /home/keaguirre-ssh/
        updatedb
        locate -n5 log

## RSync
    rsync -[options] [ssh-user-server@ip-ssh-destino:/ruta/remota/destino] [ruta/local/destino] #traer archivos desde un servidor o otra maquina
    rsync -[options] [/ruta/local/archivos] [ssh-user-destino]@[ip-ssh-destino]:[/ruta/remota/destino] #copiar datos a un server

    ej: rsync -avz keaguirre-ssh@192.168.20.144:/var/log/ /home/keaguirre-ssh/logs_backup/ #desde la maquina cliente
    /*
        -a (archive): Mantiene la estructura de directorios y conserva los atributos del archivo, como permisos y fechas.
        -v (verbose): Muestra información detallada sobre el proceso de sincronización.
        -z (compress): Comprime los datos durante la transferencia, lo que puede reducir el ancho de banda necesario.
    */

# Unidad 2

- Configuracion de un puerto en firewall de manera persistente
    -   sudo firewall-cmd --add-port=80/tcp --permanent
- Configuracion de un servicio en firewall de manera persistente
    -   sudo firewall-cmd --add-service=[service-name] --permanent
- Recargar firewall
    - sudo firewall-cmd --reload
- Listar puertos
    - sudo firewall-cmd --list-ports

## Niveles de ejecución
- Cambiar a solo texto:

        systemctl get-default
        systemctl set-default multi-user.target
        reboot
        systemctl get-default

- Volver a graphical

        systemctl get-default
        systemctl set-default graphical.target
        reboot
        systemctl get-default

## Programacion de tareas con crontab

    listar tareas de crontab:
        crontab -l
    Agregar tareas a cron: 
        crontab -e #se utiliza usualmente para tareas tareas propias del usuario
    archivo de configuracion: /etc/crontab #se utiliza para tareas globales y se necesita su, en ambos depende del contexto del uso

    programar un script:
        por orden crear el script en:
            /etc/cron.[daily, hourly, monthly, weekly]
        chmod +x /ruta/script.sh
        crontab -e o vim /etc/crontab segun se necesite
    
    forzar el cron:
        run-parts /ruta/carpeta/cron
        ej: run-parts /etc/cron.daily
            run-parts /etc/cron.horly
            tail /var/log/cron #revisar el final del archivo

## Programacion de tareas con AT

    comando a ejecutar | at 14:00 #ejecutara ese comando a las 14hrs
    comando a ejecutar | at now + 2 days #ejecutara el comando en dos dias a partir de ahora

- Listar tareas programadas: atq
- Eliminar tareas programadas: 
    - atrm [task-id] #el id es el primer digito que se lista en el comando atq

## Recuperar pass del root mediante grub

1. interrumpir el inicio del kernel moviendo las flechas en el grub
2. presionar e para editar los parametros de inicio del kernel
3. ir a la linea de comando kernel(la que empieza con linux)
4. agregar rd.break al final, con esta opcion el sistema rompe justo antes de que entregue elcontrol a initramfs al sistema real
5. presionar Ctrl + x para iniciar con los cambios en el siguiente boot
6. ya estando en la consola, mount -o remount, rw /sysroot
7. chroot /sysroot   ---> con este comando pasa a ser mi raiz de root
passwd root y te pedira la nueva password
8. touch /.autorelabel
9. exit ---> dos veces, la primera para volver al enjaulado chroot y el segundo sale de la shell de depuracion de initramfs para volver al sistema de archivos de antes de chroot
10. reboot
11. systemctl set-default graphical.target --> set entorno grafico por default

# Unidad 3 Servicios con SELinux
- Revisar status de SELinux: sestatus #Debe estar en enforcing

## Lan Segment
1. Opciones de la maquina
2. Click en la tarjeta de red o crear una nueva
3. Seleccionar LAN Segments
4. Add y le damos el nombre
5. Dentro del serv, GUI levantamos la interfaz
6. Con nmtui le damos la ip del segmento que queremos y el nombre 

## DNS
- el servicio se llama named

        dnf install -y bind bind-utils
        vim /etc/named.conf #incluir los forwarders { 8.8.8.8; 1.1.1.1; }; en la parte superior del archivo
        configurar lo siguiente dentro:
        //forward ZONE
        zone "midominio.cl" IN {
            type master;
            file "midominio.cl.db;
            allow-update { none; };
            allow-query { any; };
        };
        //backward ZONE
        zone "ip-reversa ej: 126.168.192.in-addr.arpa" IN {
            type master;
            file "midominio.cl.rev;
            allow-update { none; };
            allow-query { any; };
        }
        crear el archivo .db y .rev en la carpeta /var/named/ y copiar el contenido del ava:
        .db file:
            TTL 86400
            @ IN SOA NS server1.duocuc.cl. (
                2023110500 ;Serial
                3600 ;Refresh
                1800 ;Retry
                64800 ;Expire
                86400 ;Minimum TTL
            )
            ;Registros A
            @ IN NS server1.duocuc.cl.
            dns-primary IN A 192.168.126.136
            server1     IN A 192.168.126.136
            duocuc.cl   IN A 192.168.126.136

            ;Registros CNAME
            www         IN CNAME server1.duocuc.cl
            intranet    IN CNAME server1.duocuc.cl

            ;Registros MX Google GSuite
            @ IN MX 1  ASPMX.L.GOOGLE.COM.
            @ IN MX 5  ALT1.ASPMX.L.GOOGLE.COM.
            @ IN MX 5  ALT2.ASPMX.L.GOOGLE.COM.
            @ IN MX 10 ALT3.ASPMX.L.GOOGLE.COM.
            @ IN MX 10 ALT4.ASPMX.L.GOOGLE.COM.
        
        .rev file:
        TTL 86400
        @ IN SOA NS server1.duocuc.cl. (
            2023110500 ;Serial
            3600 ;Refresh
            1800 ;Retry
            64800 ;Expire
            86400 ;Minimum TTL
        )
        ;Datos del servidor
        @       IN NS server1.duocuc.cl.
        server1 IN A 192.168.126.136
        ;Reversa
        15  IN PTR server1.duocuc.cl
        136 IN PTR server1.duocuc.cl
    
    - Permitir el trafico del servicio dns en el firewall

            systemctl enable named --now
            firwall-cmd --add-service dns --permanent
            firewall-cmd --reload
            ls -lZ /var/named/ | grep [archivos .db y .rev]

    - Checkear servicio DNS

            nslookup server1.duocuc.cl 192.168.126.136
            nslookup www.duocuc.cl 192.168.126.136
            nslookup intranet.duocuc.cl 192.168.126.136
            nslookup 192.168.126.136 192.168.126.136 #para checkear la reversa

## VSFTPD

    dnf install -y vsftpd
    useradd -m vsftpd_user1
    passwd vsftpd_user1
    vim /etc/vsftpd/vsftpd.conf
    -> Agregar: 
        chroot_local_user=YES
        chroot_list_enable=YES
        chroot_list_file=/etc/vsftpd/chroot_list
        allow_writable_chroot=YES
        listen=YES
        listen_ipv6=NO
        pam_service_name=vsftpd
        userlist_enable=YES

    revisar que no tengo excepciones en el enjaulamiento:
        cat /etc/vsftpd/chroot_list

    Configuracion de SELinux para vsftpd
        getsetbool -a | grep ftp 
        listara por values by default:
            ftpd_full_access --> off
            ftpd_home_dir --> off
        setsebool ftpd_full_access on
        setsebool ftpd_home_dir on
        getsetbool -a | grep ftp 
        listara por values actualizados:
            ftpd_full_access --> on
            ftpd_home_dir --> on
    
    Configuracion de firewall
        firewall-cmd --add-service ftp --permanent
        firwall-cmd --reload
        systemctl enable vsftpd --now
        systemctl status vsftpd
        firewall-cmd --info-service ftp
    
    Probar el servicio:
        Entrar con filezilla servidor [ip-server] nombre usuario: vsftpd_user1 y la pass, no aparecera nada pq esta en su jaula

## Samba

    dnf install -y samba
    vim /etc/samba/smb.conf
        [directorio1]
            comment = comentario
            path = /samba/folder1
            write = yes
            read only = no
            browsable = yes
            valid users = aguirrekev
            admin users = aguirrekev

        [directorio2]
            comment = comentario
            path = /samba/folder2
            write = yes
            read only = no
            browsable = yes
            valid users = aguirrekev
            admin users = aguirrekev

    mkdir -pv /samba/folder1
    mkdir -pv /samba/folder2
    useradd -s /sbin/nologin aguirrekev
    smbpasswd -a aguirrekev
        -> ingresa la pass de smb
    chwon aguirrekev /samba/folder1
    chwon aguirrekev /samba/folder2
    systemctl enable smb --now
    firewall-cmd --add-service samba --permanent
    firewall-cmd --reload

    Actualizar contexto de carpetas para SELinux
        semanage fcontext -a -t samba_share_t '/samba'
        semanage fcontext -a -t samba_share_t '/samba/folder1'
        semanage fcontext -a -t samba_share_t '/samba/folder2'
        restorecon -v '/samba'
        restorecon -v '/samba/folder1'
        restorecon -v '/samba/folder2'
        ls -lZ / | grep samba
        ls -lZ /samba/ 
    
    Troubleshooting con SELinux
        sealert -a /var/log/audit/audit.log #Lee el error y da un resumen
        luego leer el error e ingresar los comandos que indique para la solucion de los errores junto con revisas sus advertencias
        systemctl restart smb

    Probar SMB
    en el file explorer de windows en la ruta \\192.168.126.136 para ingresar el recurso, pedira las credenciales correspondientes

## NFS

    dnf install -y nfs-utils
    mkdir -pv /shared/recurso1
    mkdir -pv /shared/recurso2
    chown nobody.nobody /shared/recurso1
    chown nobody.nobody /shared/recurso2
    vim /etc/exports
        /shared/recurso1  *(rw, sync) #en el * en vez de eso puedo especificar algun host de quien esperar el acceso a ese recurso
        /shared/recurso2  *(rw, sync)
    
    systemctl enable nfs-server --now
    firewall-cmd --list-services
    firewall-cmd --add-service rpc-bind --permanent
    firewall-cmd --add-service mountd --permanent
    firewall-cmd --add-service nfs --permanent
    firewall-cmd --reload

    En el cliente:
        showmount -e 192.168.6.2 #para revisar que recursos esta compartiendo ese host
        mkdir -pv /mounted/recurso1
        mkdir -pv /mounted/recurso2
        sudo mount -t nfs 192.168.6.2:/shared/recurso1 /mounted/recurso1
        sudo mount -t nfs 192.168.6.2:/shared/recurso2 /mounted/recurso2
        ls /mounted

# Apache
- Configuracion de apache con puerto 99

        dnf install -y httpd
        vim /etc/httpd/conf/httpd.conf
            -> ajustar en Listen a: Listen 0.0.0.0:99 # :99 porque estamos ajustando un puerto diferente al 80
        echo hola >> /var/www/html/index.html
        firewall-cmd --add-service http --permanent
        firewall-cmd --reload
        
        Asignar el puerto 99 en Selinux:
            semanage port -a -t http_port_t -p tcp 99
            systemctl enable httpd --now
            firewall-cmd --reload
            semanage port --list | grep http #Verificar que el 99 este en http_port_t
            firewall-cmd --add-port=99/tcp --permanent
            firewall-cmd --reload
        Validar desde el cliente usando la ip-server:99

# DHCP

    dnf install -y dhcp-server
    cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf #Copia el archivo de ejemplo al archivo de configuracion
    vim /etc/dhcp/dhcpd.conf configurar el siguiente bloque:
        subnet 192.168.6.0 netmask 255.255.255.0 {
            range 192.168.6.10 192.168.6.60;
            option routers 192.168.6.1;
            option subnet-mask 255.255.255.0;
            option doman-name-servers 192.168.6.2 8.8.8.8 #DNS propio luego el de google
            default-lease-time 600;
            max-lease-time 7200;
        }    
        systemctl enable dhcpd --now
        firewall-cmd --add-service dhcp --permanent
        firewall-cmd --reload
    
    Cliente dhcp (fedora): 
        dnf install -y dhcp-client
        nmcli connection modify ens160 ipv4.method auto
        nmcli connection down ens160; nmcli connection up ens160;
        hostname -I

    
