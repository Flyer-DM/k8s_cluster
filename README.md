# Разворачивание трёхнодового k8s кластера

## Описание
Kubernetes кластер - это несколько машин (компьютеров), которые работают вместе для запуска и управления приложениями в контейнерах, которые мы называем
[микросервисами](https://dzen.ru/a/X-3C568ULwsX2F_N?utm_referer=yandex.ru). Kubernetes обеспечивает 
автоматическое масштабирование, балансировку нагрузки, управление ресурсами и высокую доступность приложений в кластере.
![kubernetes](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/018a3156-c9bc-4d5b-a42b-cc4e55213d60)
Кластер будет состоять из трёх виртуальных компьютеров (нод), установленных через [VirtualBox](https://www.virtualbox.org/): мастер-нода (управляющая кластером)
и две worker-ноды, на которых непосредственно можно разворачивать приложения. Машины будут работать на ОС astra linux версии [orel](https://dl.astralinux.ru/astra/stable/orel/iso/orel-current.iso).
Устанавливать все необходимые инструменты для управления кластером будем через [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) -
опенсорсную систему для управления конфигурациями на удалённых узлах в сети. Она позволяет автоматически настраивать и развёртывать програмное обеспечение по защищённому ssh-соединению.
С такой системой пропадает необходимость вручную устанавливать необходимые компоненты для работоспособности кластера на множестве одинаковых машин, то есть не нужно будет выполнять
одни и те же действия на одинаковых компьютерах. А все необходимые конфигурации для настройки самого кластера также не придётся прописывать на yaml (языке-разметки
сценариев, который понимает ansible), все сценарии уже подготовлены в репозитории [Kubespray](https://github.com/kubernetes-sigs/kubespray) 18 версии, что соответствует
k8s-кластеру версии v1.22.5. Kubespray версии 2.22 на данный момент является новейшим, но в ходе эксперементов сочетания разного ПО выяснилось, что версия 2.18 является самой устойчивой.

P.S. Я постараюсь описать всю установку пункт за пунктом, включая возможные ошибки, которые возникали у меня, так как работа с astra linux нетривиальна, и в большинстве гайдов,
которыми я пользовался, пропущены значимые моменты, однако я также могу случайно упустить некоторые пункты.

### 0)Начальная настройка машин
Минимальные требования для хост-машины: 16 Гб ОЗУ, 100+ Гб свободного места на жёстком диске, процессор от intel core i5, доступ в сеть.

Создадим три машины:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/dcfb8fbe-9a6c-4c38-b07f-d3850ac57655)

Как вариант создать одну машину, провести все настройки до инициализации кластера, затем склонировать её два раза для создания worker-узлов (у клонов будет другой ip).

На мастер-ноде (m) установим 4 Гб ОЗУ, 4 ядра.
На рабочих нодах (w1, w2) установим по 2 Гб ОЗУ, 2 ядра (в случае клонирования просто понизить значения в настройках).
20 Гб виртуального жёсткого диска для всех трёх будет достаточно.
В настройках сети у всех трёх машин добавим второй адаптер: тип подключения - виртуальный адаптер хоста (для доступа в интернет и связывания машин в одну сеть).
На всех трёх нодах видеопамять выкрутим до 128 Мб (для настройки виртуального монитора, который по умолчанию будет в маленьком окне).
Можно также включить общий буфер обмена для всех машин.

Мастер нода:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/8fa103a3-d60b-4618-8602-6b2feb5df06b)

Рабочие ноды:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/8dd88d17-b0ea-4745-9b4c-85f675b494c6)

Включим в настройках сети "Виртуальные сети хоста" с автоматическим адаптером:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/bb9f8ef9-4582-4753-820c-d03515d2e947)
![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/3583467b-76ff-40a4-88a0-722341fb5a94)

Запускаем машины. Первоначальную установку лучше всего проводить не в графическом режиме. Вписываем имя машины, имя пользователя, пароль (не менее 8 символов). Выбираем часовой пояс.
Метод разметки - "весь диск", выбираем наш диск, схема разметки - "все файлы в одном разделе", "закончить разметку и записать изменения на диск". Ядро для установки - дефолтное (уже выбрано будет).
ПО выбираем следующее:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/a12c52d6-ab73-4e83-98ef-d6c804257e2f)

Дополнительные настройки:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/37979c50-062d-44a3-a18c-08534b9f3571)

Устанавливаем системный загрузчик GRUB, выбираем "/dev/sda", выбираем "продолжить" после сообщения об удачной загрузке. Входим в созданный ранее аккаунт.

После предустановки ОС виртуальный монитор будет в маленьком окне. Это нужно исправить. Строка меню VB -> "Устройтва" -> "Подключить образ диска Дополнительной гостевой ОС".

Открываем терминал Fly через системные утилиты:
```
sudo passwd # устанавливаем пароль администратору (должен быть сложный - цифры, буквы, длиной не менее 8)
cd /media/cdrom0
sudo su # входим под правами админа
apt update && apt-get install bzip2 tar
apt install gcc make perl linux-headers-`uname -r`
sh ./VBoxLinuxAdditions.run
shutdown -r now # перезагружаем машину
```
После перезагрузки виртульный монитор будет нормально растягиваться на весь экран, заработает общий буфер обмена.
Если вместо графической оболочки открывается консоль, то после ввода логина и пароля прописываем следующую команду:
```
startx
```
Жмём правой кнопкой по рабочему столу -> "Свойства" -> "Блокировка" -> отключим блокировку экрана:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/67065440-e6e5-4da3-a68c-3ebc475e6db5)

-> "Настройки электропитания" -> отключим "Выключение монитора", "События от кнопок", "Сон" в "Питание от сети", "Питание от батареи", "Низкий уровень заряда":

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/1a95cc59-a37b-4e97-ba7f-850938a18e79)

Данные действия с настройкой электропитания проделываем, так как наши машины являются виртуальными серверами, соотвественно уходить в сон или блокировать экран они
не должны.

Открываем консоль, заходим под администратором и подключаем репозитории со стандартными пакетами (которые разрабы astra linux решили зачем-то удалить):
```
sudo su
echo "deb https://deb.debian.org/debian/               buster         main contrib non-free" >> /etc/apt/sources.list
echo "deb https://security.debian.org/debian-security/ buster/updates main contrib non-free" >> /etc/apt/sources.list
echo "deb https://archive.debian.org/debian/ stretch main contrib non-free" >> /etc/apt/sources.list
apt update # пробуем обновить
```
Обновление неудачное:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/a75ce475-495f-4461-b2ff-658a4b6da324)

Добавляем запрашиваемые ключи:
```
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 112695A0E562B32A
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 54404762BBB6E853
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 648ACFD622F3D138
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 0E98404D386FA1D9
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys DCC9EFBF77E11517
apt update
```
Теперь списки пакетов для установки успешно обновлёны. На всех машинах узнаём ip-адрес в сети следующей командой:
```
hostname -I
```
![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/80631d45-a1bb-4fb8-b508-7e5a0056b1dc)

Их будет два, запоминаем второй. Далее редактируем файл с именами хостов следующим образом:
```
echo "192.168.56.111 m" >> /etc/hosts
echo "192.168.56.112 w1" >> /etc/hosts
echo "192.168.56.113 w2" >> /etc/hosts
```
где m - мастер нода, w1, w2 - наши рабочие ноды. 
Теперь нужно убедиться, что машины видят друг друга в сети, если уже сделали клонов. Пингуем с каждой машины каждую, выполнив следующие команды (соответственно все три машины должны быть
включены):
```
ping 192.168.56.111 -c 1
ping 192.168.56.112 -c 1
ping 192.168.56.113 -c 1
ping m -c 1
ping w1 -c 1
ping w2 -c 1
```
![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/34217396-c9e0-4eb6-84a5-bfb32cb42b35)

Как видно, каждая машина получила отправленный пакет.
Ansible и kubespray не могут работать на astra linux, потому что для нашей ОС не выпущены версии. Но программы можно обмануть, так как astra linux написана на основе ubuntu,
то следует заставить работающие на ней программы думать, что они работают на версии ubuntu. Для этого нужно поменять два системных файла: ```/etc/os-release``` и ```/etc/lsb-release```
со следующим содержимым:
```
NAME="Ubuntu"
VERSION="20.04.3 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.3 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```
и
```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.3 LTS"
```
соответственно. После чего перезагружаем машины ```shutdown -r now```. Файлы **НЕЛЬЗЯ** удалять и создавать новые с данным содержимым, их следует отредактировать через nano/vim/mcedit и т.д.
После перезагрузки отключаем файл подкачки (одно из требований работы k8s кластера):
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab # отключаем автоматическое включение после перезагрузки
free -m # проверяем, что файл выключен (последняя строка должна быть с нулевыми значениями)
```
> [!NOTE]
> Делаем снимок состояния машин (Host+T) для возможности отката.

### 1)Установка Python
Ansible написан и работает на питоне, поэтому необходимо установить Python версии 3.7.3, на которой будут работать Ansible 2.18 и его модули, а также менеджер пакетов pip
для соответственной версии Python. pip по умолчанию ставится для Python 3.5, поэтому придётся устанавливать и переустанавливать его до необходимой версии.
```
apt-get install -y curl python3.7 python3.7-dev python3.7-distutils
update-alternatives --install /usr/bin/python python /usr/bin/python3.7 1
update-alternatives --set python /usr/bin/python3.7
curl -s https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python get-pip.py --force-reinstall && rm get-pip.py
python -V # проверяем версию питона
pip -V # проверяем версию менеджера пакетов
```
![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/0265a0dc-0465-474c-96f7-89ceee6509bf)

> [!NOTE]
> Делаем снимок состояния.

### 2)Установка необходимых пакетов
Данные пакеты ставим на все ноды:
```
apt install conntrack aufs-tools software-properties-common python3-apt apt-transport-https ethtool git -y
cd /usr/lib/python3/dist-packages
# добавим ссылки для избежания возможных багов с пакетом python3-apt:
ln -s apt_inst.cpython-35m-x86_64-linux-gnu.so apt_inst.so
ln -s apt_pkg.cpython-35m-x86_64-linux-gnu.so apt_pkg.so
```
Далее нужно настроить статиченые DNS-сервера для всех машин, они требуются для будущих обновлений пакетов и установки утилит управления кластером,
скачаем и включим необходимые сервисы:
```
apt install resolvconf
systemctl enable resolvconf.service
systemctl start resolvconf.service
systemctl start systemd-resolved.service
systemctl enable systemd-resolved.service
systemctl status resolvconf.service
systemctl status systemd-resolved.service
```
Статус должен быть зелённым со значением active. Добавим в файл ```nano /etc/resolvconf/resolv.conf.d/head``` строку ```nameserver 8.8.8.8```. Обновим файл:
```
resolvconf --enable-updates
resolvconf -u # обновим добавленное значение
cat /etc/resolv.conf # проверка изменения содержимого
```
![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/56b21328-5b32-4551-b736-284afc5af519)

> [!NOTE]
> Делаем снимок состояния.

### 3)Обмен ssh ключами с рабочими нодами
Ansible будет посылать разные команды на удалённые машины, в том числе на машину, на которой он работает. Для этого необходимо удалённое подключение. Наиболее удобный
вариант это подключение через ssh-ключи без необходимости ввода пароля. Настроим такое подключение. Тут уже параллельно должны быть включены все три узла. 
Ansible не может работать от имени администратора (root), поэтому дальнейшние действия выполним через обычного пользователя **ТОЛЬКО** на управляющем узле:
```
ssh-agent bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -P "" # генерация ключа
# далее "master" - имя пользователя компьютера
ssh-copy-id -i $HOME/.ssh/id_rsa.pub master@192.168.56.112 # перессылка ключа первому воркеру
ssh-copy-id -i $HOME/.ssh/id_rsa.pub master@192.168.56.113 # перессылка ключа второму воркеру
ssh-copy-id -i $HOME/.ssh/id_rsa.pub master@192.168.56.111 # перессылка ключа мастеру (самому себе) для подключения ansible
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys  # сохранение сгенерированного ключа для управляющей ноды
chmod og-wx ~/.ssh/authorized_keys # изменение прав доступа к ключу для ansible
```
> [!NOTE]
> Делаем снимок состояния.

### 4)Установка kubespray-2.18
Установим kubespray 18 версии на управляющую ноду, пользуясь официальным руководством: https://github.com/kubernetes-sigs/kubespray.
```
cd /home/master
mkdir k8s
cd k8s
git clone --branch v2.18.0 https://github.com/kubernetes-sigs/kubespray
cd kubespray/
pip install --ignore-installed -U -r requirements.txt # установка необходимых модулей для ansible
cp -rfp inventory/sample inventory/mycluster
# прописываем IP адреса наших машин
declare -a IPS=(192.168.56.111 192.168.56.112 192.168.56.113)
CONFIG_FILE=inventory/mycluster/hosts.yaml python contrib/inventory_builder/inventory.py ${IPS[@]}
```
Далее заменим содержимое файла ```/home/master/k8s/kubespray/inventory/mycluster/inventory.ini```:
```
[all]
m ansible_host=192.168.56.111 ip=192.168.56.111
w1 ansible_host=192.168.56.112 ip=192.168.56.112
w2 ansible_host=192.168.56.113 ip=192.168.56.113


[kube-master]
m


[etcd]
m


[kube-node]
m
w1
w2


[k8s-cluster:children]
kube-master
kube-node
```
Здесь мы определили ip-адреса нод и их dns имена (all), управляющую ноду (kube-master), место положения распределённой БД файлов конфигурации для управления
кластером (etcd), рабочие ноды (kube-node), в том числе и наша мастер-нода, то есть мы разрешаем разворачивать сервисы и на управляющей ноде. 
Поменяем права доступа настроенной директории:
```
chmod 777 -R /home/master/k8s/kubespray/ # раздача полных прав доступа
```
Перед инициализацией кластера поменяем один параметр в конфигурацинном файле кластера (без изменения при инициализации может появится ошибка):
```nano /home/master/k8s/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml```
меняем ```nodelocaldns_ip: 10.233.0.10```.

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/05304dce-ba33-435c-971a-377957e1d349)

> [!NOTE]
> Делаем снимок состояния.

### 5)Инициализация кластера
Кластер сконфигурирован, теперь можно запускать установку (все узлы нашего кластера должны быть включены).
Для запуска программы развёртывания нам понадобится перейти в директорию /home/master/k8s/ и выполнить следующую команду от лица **ПОЛЬЗОВАТЕЛЯ**:
```
ansible-playbook ./kubespray/cluster.yml -i ./kubespray/inventory/mycluster/inventory.ini --become --become-user=root
```
Установка занимает 20-30 минут, скорость интернет-соединения влияет, так как будут закачиваться файлы достаточного объёма,
все логи установки будут отображаться в терминале. Красные сообщения без надписи "FATAL" гласят о нефатальных ошибках, установщик пропустит их.
В конце мы должны увидеть что-то наподобие:

![Снимок экрана 2023-08-01 120654](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/c3de3ee5-9ae0-491b-8c04-4a61084a82f2)

Если везде значение параметра "failed" равно 0, значит кластер успешно инициализирован.

### 6)Настройка управления кластера
После инициализации **НЕ ВЫКЛЮЧАТЬ** наши машины. Нужно провести некоторые настройки, чтобы после перезагрузки машин все ноды снова объединялись в кластер:
```
sudo su # заходим под администратора
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" > /etc/environment
export KUBECONFIG=/etc/kubernetes/admin.conf
mkdir -p $HOME/.kube # создание пользовательской папки
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config # копирование конфига кубера в пользовательскую папку
sudo chown $(id -u):$(id -g) $HOME/.kube/config  # изменение прав доступа
```
Теперь мы можем управлять нашим кластером от администратора.

Для начала проверим, что наши ноды соединены и готовы для работы:

![Снимок экрана 2023-08-01 120746](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/fcca27a2-1a53-4e22-9985-eaeb8cc56f6f)

Статус "READY" означает, что всё готово, он может появится не сразу, следует подождать.

Теперь посмотрим, какие "кластерные" поды уже работают. Pod (под) в k8s - это самая маленькая и базовая единица развертывания приложения.
Он представляет собой группу одного или нескольких контейнеров, которые работают вместе на одном узле и имеют общую среду выполнения.
Каждый под имеет свой уникальный IP-адрес внутри кластера и может быть настроен для доступа к различным сервисам и ресурсам. Под управляется контроллером
репликации или деплойментом и может быть автоматически масштабирован в зависимости от нагрузки на приложение.
Выполним команду ```kubectl get pods -A```:

![Снимок экрана 2023-08-01 121011](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/72efc58b-858a-4dbb-8a9d-d6df22b19222)

Видим, что три системных пода в состоянии "не готов" со статусом ошибки. Исправим эту ошибку. Для этого на всех нодах нашего кластера выполним следующие действия:
изменим определённую строку в файле ```nano /etc/kubernetes/kubelet-config.yaml``` на ```resolvConf: "/etc/resolv.conf"```

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/f29e163c-8a86-420a-8828-88ab012c507d)

> [!NOTE]
> Сделаем снимки состояния на всех машинах и перезагрузим их.

После перезагрузки машины должны соединиться в кластер. Заходим под администратором и проверяем это:

![Снимок экрана 2023-08-03 115059](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/0432fd02-38fa-4795-8659-2a58568a1241)

Все поды заработали.  *Кластер полностью настроен и готов к развёртыванию микросервисов!* 

### 7)Установка вспомогательных сервисов
Настроенные утилиты дают полный контроль над кластером, но управлять системой через консоль без графического интерфейса не всегда конфортно. Гораздо удобнее, когда все данные о состоянии компонентов кластера отображаются в реальном времени с возможностью настройки конфигураций несколькими кликами без необходимости ввода множества команд в терминале linux. Для этого установим два вспомогательных сервиса: [dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) - графической веб-интерфейс и [metrics-server](https://github.com/kubernetes-sigs/metrics-server) для отслеживания показателей ресурсов нашего кластера. Перейдём к установке. Выполним следующую команду:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
Это команда автоматически скачает образ микросервиса dashboard и запустит его на нашем кластере. Но мы не будем иметь доступ к сервису, то есть не сможем открыть приложение в браузере, так как по умолчанию сервис настроен как внутрикластерный, чтобы только компоненты кластера могли как-либо взаимодействовать с его ресурсами. Нужно поменять тип сервиса на NodePort, тогда k8s назначит один свободный порт из диапазона 30000-32767. После чего обратиться к сервису можно будет по ip адресу ноды, на которой он работает и номеру выделенного порта. Отредактируем файл конфигураций dashboard следующей командой:
```
kubectl -n kube-system edit service kubernetes-dashboard -n kubernetes-dashboard
```
И поменяем type на NodePort:

![Снимок экрана 2023-08-03 122914](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/d21b9c25-b274-4d55-a2a1-9636a56d234c)

Кубер автоматически обновит конфиг файл и перезапустит сервис. Доступ к нему появится не сразу, через 1-3 минуты. Пока настроим админский аккаунт для генерации секретного ключа для доступа к dashboard. Создадим файл ```nano dashboard_acc.yaml``` и пропишем следующее:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
Применим конфиг файлы:
```
kubectl apply -f dashboard_acc.yaml
```
Нужно сгенерировать токен для досутпа к сервису:
```
kubectl -n kube-system create token admin-user
```
Токен напечатается в консоль, копируем его. Также нужно узнать , на какой ноде запущен наш сервис и на каком порте:
```
kubectl get pods -A -o wide
kubectl get services --all-namespaces
```

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/a37070ee-2ca2-4569-aaae-0e52bf4d0585)

В нашем случае сервис запущен на ноде "m" с ip 192.168.56.111 на порту 30087. Открываем браузер и печатаем: "https://192.168.56.111:30087/".
Жмём "Подробности" -> "Сделать исключение для этого сайта". Наш dashboard открылся:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/59dae37c-de07-4776-84eb-fcabca151b46)

Вставляем ранее скопированный токен и получаем доступ к дашборду нашего кластера.

Теперь настроим metrics-server. Возвращаемся в терминал и скачиваем образ:
```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml
```
Редактируем файл: ```nano high-availability-1.21+.yaml```: добавляем строчку ```- --kubelet-insecure-tls``` в раздел "args":

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/17607be5-3884-4f6f-adf4-c8a01133bf11)

Запускаем сервис:

```
kubectl apply -f high-availability-1.21+.yaml
```
Ждём 1-2 минуты и теперь мы можем получать информацию по нагрузке на наш кластер, в том числе в графическом виде в dashboard в виде графиков:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/015393b5-2a60-4820-9039-aa582b9c27dd)

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/5542142e-2f7c-4ce8-b249-2879fe8438c0)

Установим дополнительный плагин [k9s](https://k9scli.io/) для мониторинга кластера прямо в терминале:
```
curl -sS https://webinstall.dev/k9s | bash
source ~/.config/envman/PATH.env
```
Запустим плагин командой ```k9s``` в отдельном окне терминала:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/656ff0f8-655d-4280-bba0-a32f85771de0)

А чтобы добавлять встроенные в kubernetes плагины нужно установить менеджер [krew](https://krew.sigs.k8s.io/). Создадим файл ```nano install_krew.sh``` с содержимым:
```
set -x; cd "$(mktemp -d)" &&
curl -fsSLO "https://github.com/kubernetes-sigs/krew/" &&
tar zxvf krew.tar.gz &&
KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&
"$KREW" install krew
```
Сделаем его исполняемым и запустим:
```
chmod +x install_krew.sh
./install_krew.sh
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH # установим переменную окружения PATH"
```
Теперь для установки дополнительных плагинов можно прописывать команду ```kubectl krew install <имя-расширения>```.

> [!NOTE]
> Сделаем снимок состояния мастер-ноды.

### 8)Запуск микросервиса с открытым образом
Установим микросервис с открытым образом. Для примера возьмём [tomcat](https://tomcat.apache.org/) - веб-сервер для отладки, тестирования и запуска веб-приложений на java с открытым исходным кодом, содержащий ряд программ самоконфигурации. Перед установкой необходимо скачать исходный архив из раздела Core с файлами конфигурации для настройки пользователя c официального сайта: https://tomcat.apache.org/download-80.cgi. Распакуем в пользовательскую папку home/master.
Добавим пользователя, для управление веб-сервисом. Для этого поменяем содержимое файла (можно удалить и создать новый с тем же именем) ```nano /home/master/tomcat-conf/conf/tomcat-users.xml``` на:
```
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
      version="1.0">
      <role rolename="manager-gui"/>
      <role rolename="admin-gui"/>
      <user username="tomcat" password="tomcat" roles="manager-gui,admin-gui"/>
</tomcat-users>
```
Добавили пользователя с именем и паролем tomcat. Но при запуске приложения этот файл нужно подменить внутри самого контейнера. Для этого добавим пути монтирования в yaml файл деплоймента приложения. Создадим файл nano ```tomcat.yaml``` с содержимым:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  replicas: 1
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat-container
        image: tomcat:8.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: tomcat-t1
          mountPath: /usr/local/tomcat/conf/ # монтируемый путь внутри контейнера
      volumes:
      - name: tomcat-t1
        hostPath:
          path: /home/master/tomcat-conf/conf/ # монтируемый путь на ноде
      - name: tomcat-t1
        configMap:
        name: tomcat-config
---
apiVersion: apps/v1
kind: Service
apiVersion: v1
metadata:
  name: tomcat-service
spec:
  type: LoadBalancer
  selector:
    app: tomcat
  ports:
  - name: http
    protocol: TCP
    port: 8080
```
Теперь нужно создать два объекта для управления хранилищем. Добавим объект хранилища PersistentVolume ёмкостью 1 Гб с режимом доступа ReadWriteOnce, позволяя только одному узлу монтировать этот объект для чтения и записи данных, и PersistentVolumeClaim, представляющий собой запрос модуля доступа к данным.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: tomcat-t1
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/master/"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tomcat-t1
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
```
Создадим наши хранилище и запустим сервис (нужно подождать пару минут, когда контейнер инициализируется и запустится):
```
kubectl create namespace tomcat-ns
kubectl apply -f tomcat-pv-pvc.yaml
kubectl apply -f tomcat.yaml -n tomcat-ns # разворачиваем сервис в созданном namespace
```
Контейнер заработал на мастер ноде:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/7d4d4c31-f066-4f8d-9651-88feac54ace0)

Узнаём порт, на котором тот работаем и открываем веб-сервис:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/e6eb0119-0680-4797-9fca-53a17c603e7b)

Проверим наш аккаунт. Для этого открываем, например, "Server Status", вводим наш логин и пароль. Если открывается страница, то всё правильно настроенно:

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/1a72d1b9-7128-43ab-be42-66df6f3fcb81)

Наш вмонтированный конфиг файл можно посмотреть, открыв консоль самого контейнера в dashboard (Exec):

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/c899b733-3d02-4316-8deb-0dff7d94b4a4)

![image](https://github.com/Flyer-DM/k8s_cluster/assets/113033685/81123d17-21c1-45a6-9d01-a20d73a0c908)

Планировщик scheduler распределил под с tomcat сервисом на ноду m, так как на неё в момент запуска контейнера была самая низкая нагрузка. Но, если бы под был развёрнут на других нодах, то приложение не получило бы доступ к вмонтированной директории, потому что этой директории на других нодах просто нет. Это нужно исправить. Для этого нужно настроить резервное копирование файлов конфигурации с мастер ноды на все остальные, при этом оно должно быть автоматическим и с постоянным обновлением, ведь файлы конфигурации могут поменяться, а вручную менять их на всех узлах кластера неудобно. Для это настроим безопасную перессылку файлов через утилиту scp с постоянным исполнением через cron. Отредактируем файл ```crontab -e```, добавив в конец строки:
```
*/5 * * * * scp -r /home/master/tomcat-conf/ master@w1:/home/master/
*/5 * * * * scp -r /home/master/tomcat-conf/ master@w2:/home/master/
```
Теперь каждые 5 минут наша директория с конфигами tomcat будет перессылаться на рабочие ноды в ту же директорию.

 *Так мы запустили наш первые микросерсвис* 
