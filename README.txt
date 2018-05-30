# aspnet-core-ci-docker
Instruction for https://youtu.be/nGG-rd2JYXk

firewall
======================

sudo su

Проверим, запущен ли firewalld
	systemctl status firewalld
	
Запуск firewalld
	systemctl start firewalld
	
Включение автостарта
	systemctl enable firewalld
	
Список всех активных зон
	firewall-cmd --get-active-zones
	----
		public
		  interfaces: eth0	
		  
Смена установление сетевого интерфейса если отсутсвует		  
	firewall-cmd --permanent --zone=public --change-interface=eth0	
	systemctl restart firewalld
	
Убрать добавить необходимые сервисы и порта
	firewall-cmd --permanent --zone=public --remove-service=dhcpv6-client
	firewall-cmd --permanent --zone=public --add-port=2234/tcp
	firewall-cmd --permanent --zone=public --add-service=http
	firewall-cmd --permanent --zone=public --add-service=https
	
Все доступные сервисы
	firewall-cmd --get-service
	
Обобразить открытые порта
	firewall-cmd --zone=public --list-services
	firewall-cmd --zone=public --list-ports
	
Что разрешено постоянно на нашем сервере	
	firewall-cmd --permanent --list-all
		  
docker
======================
https://docs.docker.com/engine/installation/linux/docker-ce/centos/#install-docker-ce

sudo su

Установка необходимых пакетов
	yum install -y yum-utils device-mapper-persistent-data lvm2
	
Добавление docker репозитория
	yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
	
Установка docker
	yum install docker-ce
	
https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/
Смена storage-driver
	mkdir /etc/docker
	cat  >> /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "storage-opts": [
	"overlay2.override_kernel_check=true"
  ]
}

Проверить после старта storage-driver
	docker info
	
Использование docker без sudo
Может привести к потанциальной опасности так как docker группа имеет права эквивалентные root пользователю	
https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface

Создаем docker группу
	groupadd docker

Добавляем пользователя в группу
	usermod -aG docker $USER
	
Запуск docker
	systemctl start docker
	
Включение автостарта
	systemctl enable docker

Выйдем и зайдем заново чтобы применилась групповое членство	

Проверим докер
	docker run --rm hello-world
	
Enviroment
==================

Создание виртуальной сети в докере
	docker network create smarly
	
Запуск nginx

Создадим папки для маппинга
	mkdir ~/docker
	mkdir ~/docker/nginx
	cd docker/nginx/
	mkdir conf.d
	mkdir vhost.d
	mkdir certs
	mkdir www
	
Скопируем конфиг и стартовую страницу для теста с докер образа в шареную папку
	docker run --name nginx-temp -d nginx
	docker cp nginx-temp:/etc/nginx/nginx.conf ~/docker/nginx/nginx.conf
	docker cp nginx-temp:/etc/nginx/conf.d ~/docker/nginx
	docker cp nginx-temp:/usr/share/nginx/html ~/docker/nginx/www
	docker container stop nginx-temp
	docker container rm nginx-temp

Создаем и запускаем nginx	
	docker run --detach --network smarly --restart always --name docker-nginx \
	--publish 443:443 --publish 80:80 \
	-v ~/docker/nginx/conf.d:/etc/nginx/conf.d  \
	-v ~/docker/nginx/vhost.d:/etc/nginx/vhost.d  \
	-v ~/docker/nginx/nginx.conf:/etc/nginx/nginx.conf  \
	-v ~/docker/nginx/www:/usr/share/nginx \
	-v ~/docker/nginx/certs:/etc/nginx/certs:ro \
	nginx

Добавляем возможность загрузки больших файлов
	vim ~/docker/nginx/nginx.conf
		http {
			...
			# disable any limits to avoid HTTP 413 for large image uploads
			client_max_body_size 0;
			...
		}	
	
Перегружаем сервис
	docker exec -ti docker-nginx /bin/bash
	service nginx restart
	#docker kill --signal=HUP docker-nginx
	
Проверяем 
	curl localhost

Let's encript 
====================

Скачать шаблонный файл конфига
	mkdir ~/docker/docker-gen/
	curl https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl > ~/docker/docker-gen/nginx.tmpl

Запустить контейнеры
	docker run --detach --network smarly --restart always --name docker-gen \
	--volumes-from docker-nginx \
	-v ~/docker/docker-gen/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro \
	-v /var/run/docker.sock:/tmp/docker.sock:ro \
	jwilder/docker-gen \
	-notify-sighup docker-nginx -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf

	docker run --detach --network smarly --restart always --name docker-nginx-letsencrypt \
	--volumes-from docker-nginx \
	-v ~/docker/nginx/certs:/etc/nginx/certs:rw \
	-v /var/run/docker.sock:/var/run/docker.sock:ro \
	-e NGINX_DOCKER_GEN_CONTAINER=docker-gen \
	-e NGINX_PROXY_CONTAINER=docker-nginx \
	jrcs/letsencrypt-nginx-proxy-companion
	
Portainer
====================

	mkdir ~/docker/portainer
	
	openssl rand -base64 14 > /tmp/portainer_password

docker run --detach --network smarly --restart always --name docker-portainer \
-p 9000:9000   \
-v /var/run/docker.sock:/var/run/docker.sock   \
-v ~/docker/portainer:/data   \
-e "VIRTUAL_HOST=portainer.smarly.net" \
-e "LETSENCRYPT_HOST=portainer.smarly.net" \
-e "LETSENCRYPT_EMAIL=smarly@smarly.net" \
portainer/portainer

#https://portainer.smarly.net/#/init/admin
login admin
pass Ig2g10ZaGDY4r7AtL4A=

Выбрать Local

gitlab
======================

	mkdir ~/docker/gitlab
	cd ~/docker/gitlab
	mkdir config
	mkdir logs
	mkdir data
		
	docker run --detach \
	--name docker-gitlab \
	--network smarly  \
	--publish :80 \
	--restart always \
	--volume ~/docker/gitlab/config:/etc/gitlab \
	--volume ~/docker/gitlab/logs:/var/log/gitlab \
	--volume ~/docker/gitlab/data:/var/opt/gitlab \
	-e "VIRTUAL_HOST=gitlab.smarly.net" \
	-e "LETSENCRYPT_HOST=gitlab.smarly.net" \
	-e "LETSENCRYPT_EMAIL=smarly@smarly.net" \
	gitlab/gitlab-ce:latest

Настроить конфигурацию
	docker exec -ti docker-gitlab /bin/bash
	
	vim /etc/gitlab/gitlab.rb
		external_url 'https://gitlab.smarly.net/'
		nginx['listen_port'] = 80
		nginx['listen_https'] = false
		nginx['proxy_set_headers'] = {
		#  "Host" => "$http_host_with_default",
		#  "X-Real-IP" => "$remote_addr",
		#  "X-Forwarded-For" => "$proxy_add_x_forwarded_for",
		  "X-Forwarded-Proto" => "https",
		  "X-Forwarded-Ssl" => "on",
		#  "Upgrade" => "$http_upgrade",
		  "Connection" => "$connection_upgrade"
		}
		
	gitlab-ctl reconfigure
	
https://gitlab.smarly.net/users/password/edit?reset_password_token=NRKG2t3A3evy61ef2YhS
login root
pass NM4QRsQVjd79IogYTyQ=

gitlab runner docker-in-docker
================================
mkdir ~/docker/gitlab-runner
mkdir ~/docker/gitlab-runner/config

docker run -d --name docker-gitlab-runner --restart always --network smarly \
  -v ~/docker/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest


docker exec -ti docker-gitlab-runner /bin/bash  
  
gitlab-runner register -n \
  --url https://gitlab.smarly.net/ \
  --registration-token oUNsZ91sBYwmGZeQUExA \
  --executor docker \
  --description "Docker Runner" \
  --docker-image "docker:latest" \
  --docker-privileged

					root@9ef8fe5ef33c:/# cat /etc/gitlab-runner/config.toml
					concurrent = 1
					check_interval = 0

					[[runners]]
					  name = "Docker Runner"
					  url = "https://gitlab.smarly.net/"
					  token = "bd63fff856050f82e5add831ceed5b"
					  executor = "docker"
					  [runners.docker]
						tls_verify = false
						image = "docker:latest"
						privileged = true
						disable_cache = false
						volumes = ["/cache"]
						shm_size = 0
					  [runners.cache]
					  
  
Unlock executor

.gitlab-ci.yml

services:
  - docker:dind

stages:
  - build

build:
  stage: build
  script:
    - docker run --rm hello-world


Registry
===========================

	mkdir ~/docker/docker-registry

	docker run --detach --network smarly --restart always --name docker-registry -p 5000:5000 \
	-v ~/docker/docker-registry:/var/lib/registry \
	-e "VIRTUAL_HOST=docker-registry.smarly.net" \
	-e "LETSENCRYPT_HOST=docker-registry.smarly.net" \
	-e "LETSENCRYPT_EMAIL=smarly@smarly.net" \
	registry:2	
	
docker login docker-registry.smarly.net -u root	

	docker exec -ti docker-gitlab /bin/bash
	vim /etc/gitlab/gitlab.rb
		registry_external_url 'https://docker-registry.smarly.net'
		gitlab_rails['registry_host'] = "docker-registry.smarly.net"

		registry_nginx['listen_port'] = 80
		registry_nginx['listen_https'] = false

		registry_nginx['proxy_set_headers'] = {
		#	"Host" => "$http_host",
		#	"X-Real-IP" => "$remote_addr",
		#	"X-Forwarded-For" => "$proxy_add_x_forwarded_for",
			"X-Forwarded-Proto" => "https",
			"X-Forwarded-Ssl" => "on"
		}
		
		
	gitlab-ctl reconfigure
	
Использование своего registry сервера для сборки образа
=====================================

docker pull docker:dind
docker pull microsoft/dotnet:2.0.3-runtime
docker pull microsoft/dotnet:2.0.3-runtime

docker tag docker:dind docker-registry.smarly.net/docker-ci:dind
docker tag microsoft/dotnet:2.0.3-sdk docker-registry.smarly.net/microsoft/dotnet:2.0.3-sdk
docker tag microsoft/dotnet:2.0.3-runtime docker-registry.smarly.net/microsoft/dotnet:2.0.3-runtime

docker push docker-registry.smarly.net/docker-ci:dind
docker push docker-registry.smarly.net/microsoft/dotnet:2.0.3-sdk
docker push docker-registry.smarly.net/microsoft/dotnet:2.0.3-runtime	
	
========================

docker run --detach --network smarly --ip 172.18.0.21 --restart always --name docker-elasticsearch \
-p 9200:9200  \
-e "VIRTUAL_PORT=9200" \
-e "VIRTUAL_HOST=elasticsearch.smarly.com" \
-e "LETSENCRYPT_HOST=elasticsearch.smarly.com" \
-e "LETSENCRYPT_EMAIL=smarly@smarly.net" \
elasticsearch

docker run --detach --network smarly --ip 172.18.0.22 --restart always --name docker-kibana \
-e "ELASTICSEARCH_URL=http://172.18.0.21:9200" \
-e "VIRTUAL_HOST=kibana.smarly.com" \
-e "LETSENCRYPT_HOST=kibana.smarly.com" \
-e "LETSENCRYPT_EMAIL=smarly@smarly.net" \
kibana
