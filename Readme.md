# **Принцип работы протоколов HTTP, HTTPS, HTTP2.**
## Используемые настройки:
+ Виртуальная машина с Ubuntu: Адаптер 1 - NAT и Адаптер 2 - Internal Network (10.0.0.1 - вручную)
+ Виртуальная машина с Kali: Адаптер 1 - NAT и Адаптер 2 - Internal Network (10.0.0.2 - вручную)
+ Виртуальная машина с Windows: Адаптер 1 - NAT и Адаптер 2 - Internal Network (10.0.0.3 - вручную)
### **Установка и работа Nginx**
На этом этапе мы посмотрим, как настраивается самый популярный HTTP-сервер - nginx, сгенерируем и установим туда самоподписанный сертификат (это типичная задача внутри организаций - нужно обеспечить защищённый канал, но покупать сертификат для целей разработки или внутреннего обмена желания нет).
#### **Установка пакетов**

Все действия из этого этапа выполняются на машине с Ubuntu.

1. Откройте терминал и выполните следующие команды для установки необходимых /пакетов:
```
sudo apt update
sudo apt install nginx openssl mc vim
```
2. Поcле установки nginx автоматически запустится на 80-м порту. Нас будут интересовать HTTP (80) и HTTPS (443). Откройте их на файерволле, если он у вас включен:

![ufw](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/10.0.0.1_80%20(2).png?raw=true)


3. Проверить статус nginx с помощью команды:
```
systemctl status nginx
```
4. Убедитесь с машины с Kali, что HTTP сервер на Ubuntu открывается на 80 порту (для этого в браузере откройте http://10.0.0.1 - браузер автоматически обратиться по порту 80):
![page](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/10.0.0.1_80%20(1).png?raw=true)



### **Настройка и генерация сертификатов**
1. Нам необходимо перейти в каталог /etc, в котором будут храниться наши сертификаты
```
cd /etc/nginx
sudo mkdir certs
cd certs
```
2. Далее сгенерируем самоподписанный сертификат с помощью OpenSSL
```
sudo openssl req -newkey rsa:2048 -nodes -x509 -days 365 -keyout key.pem -out certificate.pem -subj "/C=RU/ST=Moscow/L=Moscow/O=Security/OU=Security/CN=netology.local" -addext "subjectAltName=DNS:netology.local"
```
Данная команда генерирует сертификат и ключ с заполненными данными.

В итоге в вашем каталоге должно появиться два файла:

```
certificate.pem
key.pem
```
3. Далее необходимо настроить конфигурационный файл nginx. Для этого испольуем - Vim
```
sudo vim /etc/nginx/sites-enabled/default
```
Вы увидите следующую картинку:
![vim](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/default-before.png?raw=true)
4. Вам нужно отредактировать имеющуюся конфигурацию, чтобы получить следующую (комментарии (те что начинаются с #) удалены для краткости):
```
server {
  # имя нашего сервера
  server_name netology.local;

  # слушаем на 443 порту
  listen 443 ssl;
  listen [::]:443 ssl;
  # пути к сертификату и ключу
  ssl_certificate /etc/nginx/certs/certificate.pem;
  ssl_certificate_key /etc/nginx/certs/key.pem;
  # какие протоколы поддерживаем
  ssl_protocols TLSv1.3;

  # где искать файлы, выдаваемые пользователю
  root /var/www/html;

  # какой файл выдавать по умолчанию
  index index.html index.htm index.nginx-debian.html;

  location / {
    try_files $uri $uri/ =404; 
  }     
}

server {
  server_name netology.local;

  if ($host = netology.local) {
    return 301 https://$host$request_uri;
  }

  listen 80;
  listen [::]:80;

  return 404;
}
```
5. Проверяем конфигурацию на отсутствие ошибок с помощью команды:
```
sudo nginx -t
```
Если вывод соответствует нижепредставленному, то можно двигаться дальше (если нет, читаете ошибку и исправляете):

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
6. Загружаем новую конфигурацию nginx:
```
sudo nginx -s reload
```
### **Настройка Kali**
Теперь мы хотим по домену ~~netology.local~~ обращаться к тому серверу, который работает по адресу 10.0.0.1.
1. Самый простой способ для этого (чтобы не поднимать DNS) - это отредактировать файл /etc/hosts.
Приведите его к следующему виду:
```
127.0.0.1    localhost
127.0.1.1    kali
10.0.0.1     netology.local

# Добавлена 3 строка
```
2. В браузере откройте ~~https://netology.local~~ (и нажмите на кнопку `Advanced`):
![sertificate](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/self-signed.png?raw=true)

3. Нам говорят, что сертификат соответствует домену, но не является доверенным, поскольку является самоподписанным.

Если нажать на `View Certificate`:
![sertificate](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/certificate.png?raw=true)

Теперь проверим часть, которую мы настроили в нашем конфиге для перехода с http на https с кодом (301):
```
server {
  server_name netology.local;

  if ($host = netology.local) {
    return 301 https://$host$request_uri;
  }

  listen 80;
  listen [::]:80;

  return 404;
}
```
Введите адрес ~~http://netology.local~~ и удостоверьтесь, что вас перебрасывает на ~~https://netology.local~~ (можете посмотреть в консоли разработчика - F12):
![wireshark](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/redirect.png?raw=true)


Теперь, если вы нажмёте на ```Accept Rist and Continue```, браузер вам загрузит данные по HTTPS, что вы можете увидеть в WireShark:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/tls.png?raw=true)



Посмотреть сертификат можно в адресной строке:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/cert01.png?raw=true)

Нажмите ```More Information```:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/cert02.png?raw=true)


Далее ```View Certificate```:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/cert03.png?raw=true)


И даже его можно экспортировать вкладка ```Details``` и кнопка ```Export```:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/cert04.png?raw=true)

### **HTTP2**
1. Для включения HTTP2 достаточно в конфигурацию (на Ubuntu файл /etc/nginx/sites-enabled/default) включить ``http2``:
```
server {
  server_name netology.local;

  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  # дальше без изменений
  ```

2. Сохраняете файл, тестируете (``sudo nginx -t``), перезагружаете конфигурацию (``sudo nginx -s reload``).

3. В Kali удостоверяетесь, что вам теперь страничка отдаётся по HTTP2:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/cert06.png?raw=true)


#### **Настройка Windows**
1. В Windows кликаете в таскбаре на иконке сетевого подключения:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/win01.png?raw=true)

2. Далее находите подключение и настраиваете его свойства:
3. ![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/win02.png?raw=true)

4. Если вдруг второй сетевой адаптер не настраивается, то вы можете настроить его из командной строки, запустив её от имени администратора:
```
netsh interface ip set address <имя адаптера> static 10.0.0.3 255.255.255.0
```
1. Имя адаптера можно посмотреть с помощью команды ipconfig:
![ipconfig](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/ipconfig.png?raw=true)

2. Соответственно, имя второго адаптера Ethernet 2 (с пробелом, поэтому нужно брать в кавычки):
```
netsh interface ip set address "Ethernet 2" static 10.0.0.3 255.255.255.0
```

1. После этого удостоверьтесь с помощью ``ipconfig``, что адрес изменился.

На всякий случай напоминаем, что ``etc/hosts`` в Windows находится в ``%SystemDirectory%\System32\Drivers\etc\hosts`` (например, ``C:\Windows\System32\etc\hosts``).

#### **Доверенные самоподписанные сертификаты в Windows**

1. В Windows: открываете консоль, запускаете команду certmgr:
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/certmgr.png?raw=true)

2. А дальше просто импортируете экспортированный сертификат в корневые хранилища.

3. После этого необходимо перезапустить браузер (закрыть все окна и заново открыть):
![](https://github.com/PopChop88/-/blob/76f542c18c4a9363263e4e87df05e727bb0bceb6/pic/win-trusted.png?raw=true)