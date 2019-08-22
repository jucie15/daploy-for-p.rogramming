# 장고 배포하기 실습

![서버 동작 과정](imgs/web-server-process.png)

서버 동작 과정

## Refer

1. [윈도우에서 SSH 접속하기](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/putty.html)
2. [장고 배포 체크리스트](https://docs.djangoproject.com/en/2.2/howto/deployment/checklist/)
3. [Ubuntu16.04, Nginx, Uwsgi환경에서 Django 프로젝트 배포하기](https://www.digitalocean.com/community/tutorials/how-to-serve-django-applications-with-uwsgi-and-nginx-on-ubuntu-16-04)
4. [Ubuntu16.04에서 장고 배포 후  Postgresql 사용하기](https://www.digitalocean.com/community/tutorials/how-to-use-postgresql-with-your-django-application-on-ubuntu-16-04)



## **시스템 환경**

- **Ubuntu 18.04**
- **Django 2.2**
- **nginx**
- **uwsgi**



## **Step**

1. **AWS EC2 인스턴스 생성**
  
    1. Elastic IP 연결
    2. Privatekey 생성 (윈도우는 `puttyGen`이용, 맥은 `chmod` 400)
    
2. **장고 배포 준비**
    1. 개발하는 환경에 따라 `settings`파일과 패키지 `requirements`파일을 분리하여 사용한다
        1. `requirements` 파일 분기
            1. `common.txt` ← 로컬, 개발, 상용 서버에 모두 사용되는 패키지들 모음
            2. `prod.txt` ← 상용 서버에서만 사용되는 패키지들 모음
            3. `dev.txt` ← 개발환경에서 사용되는 패키지들 모음
        2. `settings` 파일 분리 
            1. `common.py` ← 로컬, 개발, 상용 서버에 모두 사용되는 설정 모음
            2. `prod.py` ← 상용 서버에서만 사용되는 설정 모음
            3. `dev.py` ← 개발환경에서 사용되는 설정 모음
    2. `manage.py` , `uwsgi.py` 의 셋팅 모듈 값 변경
    3. `Static`, `Media` 파일 경로 설정
    4. `BASE_DIR` 경로 수정
    
3. **RDS 생성**

4. **SSH 접속**

   ```shell
    $ ssh ubuntu@ip -i ~/.ssh/[private_key] 
   ```

5. **기본 스크립트 설정 및 가상환경 준비**

   ```shell
    # 기본적인 명령어 alias 등록
    $ echo "alias python='python3'" >> ~/.bashrc
    $ echo "alias pip='pip3'" >> ~/.bashrc
    $ source ~/.bashrc # 쉘 스크립트 적용
    
    # 우분투 패키지 설치
    $ sudo apt-get update
    $ sudo apt-get install python3-pip python3-dev libpq-dev
    $ sudo apt-get install python3-venv
    $ sudo pip3 install --upgrade pip
    
    # 장고 코드 받기
    $ git clone https://github.com/jucie15/deploy-for-p.rogramming.git
    $ cd deploy-for-p.rogramming
    
    # 가상환경 생성 및 실행
    $ python -m venv [venv_name]
    $ source [venv_name]/bin/activate
    
    # 파이썬 패키지 설치
    $ pip install -r reqs/prod.txt
    
    # static 파일 서빙을 위해
    $ python manage.py collectstatic
    
    # 패키지 설치 후 uWSGI 서버 테스트
    $ uwsgi --http :8080 --home /home/ubuntu/deploy-for-p.rogramming/venv --chdir /home/ubuntu/deploy-for-p.rogramming/mysite -w mysite.wsgi
   ```

6. **uWSGI 설정**
  
    1. **uWSGI 옵션 파일(.ini) 추가**
    
       1. [uWSGI options](https://uwsgi-docs.readthedocs.io/en/latest/Options.html)
    
       ```shell
       $ sudo mkdir -p /etc/uwsgi/sites # -p 옵션은 중간에 없는 디렉토리까지 같이 생성해준다.
       $ sudo vi /etc/uwsgi/sites/[project].ini
       
       # dir: /etc/uwsgi/sites/[project].ini -> 한 서버에 여러 Django 앱을 둘 경우(Emperor 모드 사용 가능) 
       # dir: /home/[user]/run/uwsgi/[project].ini -> 하나만 있을 경우 run 폴더에 바로 설정해도 무관 
       
       [uwsgi]
       uid = ubuntu
       base = /home/%(uid)/deploy-for-p.rogramming
       project = mysite
       
       home = %(base)/venv
       chdir = %(base)/%(project)
       module = %(project).wsgi:application
       
       master = true
       processes = 5
       
       socket = /run/uwsgi/%(project).sock
       chown-socket = %(uid):www-data
       chmod-socket = 660
       vacuum = true
       
       env = DJANGO_SETTINGS_MODULE=mysite.settings.prod
       env = DB_NAME=
       env = DB_USER=
       env = DB_PW=
       env = DB_HOST=
       env = SECRET_KEY=
       ```
    
    2. **uWSGI에 대한 서비스 스크립트 생성**
    
       1. [Systemd](https://uwsgi-docs.readthedocs.io/en/latest/Systemd.html)
       2. [Systemd options](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
    
       ```shell
       $ sudo vi /etc/systemd/system/uwsgi.service
       
       [Unit]
       Description=uWSGI Emperor service
       
       [Service]
       Environment = LANG=ko_KR.utf8
       Environment = LC_ALL=ko_KR.utf8
       Environment = LC_LANG=ko_KR.utf8
       
       ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown ubuntu:www-data /run/uwsgi'
       ExecStart=/home/ubuntu/deploy-for-p.rogramming/venv/bin/uwsgi --emperor /etc/uwsgi/sites
       Restart=always
       KillSignal=SIGQUIT
       Type=notify
       NotifyAccess=all
       
       [Install]
       WantedBy=multi-user.target
       ```
    
    3. **uWSGI 서비스 등록 및 구동 확인**
    
       ```shell
       $ sudo systemctl enable uwsgi
       $ sudo systemctl start uwsgi
       $ sudo systemctl status uwsgi
       ```
    
7. **nginx 설정**

    1. **server-block 설정 파일 추가**

        ```shell
         $ sudo apt-get install nginx
         $ sudo vi /etc/nginx/sites-available/[project_name]
         
         server {
             listen 80;
             server_name [IP OR DOMAIN]; # 서버 도메인 or ip 추가
         
             location = /favicon.ico { access_log off; log_not_found off; }
             location /static/ {
                 root /home/ubuntu/deploy-for-p.rogramming/mysite;
             }
         
             location / {
                 include         uwsgi_params;
                 uwsgi_pass      unix:/run/uwsgi/mysite.sock;
             }
         }
        ```
    
    2. **설정 파일 심볼릭 링크 연결**
    
        ```shell
        $ sudo ln -s /etc/nginx/sites-available/[project] /etc/nginx/sites-enabled
        ```
    
    3. **구문 오류 체크 후 nginx 재시작**
    
       ```shell
       $ sudo nginx -t
       $ sudo systemctl restart nginx
       $ sudo systemctl enable nginx
       ```
    
    4. **방화벽 액세스 허용**
    
       ```shell
       $ sudo ufw allow 'Nginx Full'
       ```
    
8. **SSL/ TLS를 사용 트래픽 보호하기**