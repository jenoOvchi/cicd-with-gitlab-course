# Технология непрерывной поставки ПО

## Topic 1: Install Gitlab

Официальная инструкция по установке  Gitlab на Google Cloud Platform  доступна по следующей ссылке - https://docs.gitlab.com/ee/install/google_cloud_platform/

Регистрируемся в Google Cloud Platform. Для этого:
1. Переходим на сайт GCP (https://cloud.google.com/) и нажимаем Get started for free.
2. Вводим свои данные и карточку. Деньги с карточки не будут списываться, если вы сами не активируете платную подписку.

После регистрации вы окажетесь на главной странице сервиса. Вам необходимо выбрать в разделе Ресурсов вкладку Compute Engine. Необходимо создать новый экземпляр для установки GitLab. Для этого нажмём кнопку "Создать экземпляр". Требуется задать имя виртуальной машины (gitlab) и зону её размещения (us-central1-c). Для GitLab рекомендуется 2 vCPU и 4 GB RAM, поэтому выбираем тип VM "n1-standart-2". Необходимо будет выбрать также ОС и размер доступного диска. В разделе "Загрузочный диск" нажимаем кнопку "Изменить", выбираем Ubuntu 16.04 LTS, в разделе "Тип загрузочного диска" выбираем "Постоянный SSD-диск", указываем размер "40 ГБ" и нажимаем кнопку "Выбрать". После этого в разделе "Брандмауэр" ставим флаги "Разрешить трафик HTTP" и "Разрешить трафик HTTPS" и нажимаем кнопку "Создать". Всё, ВМ создана. Обычно её развертывание занимает от 1 до 5 минут. После создания виртуальной машины обращаем внимание на значение в столбце "Внешний IP-адрес" - в дальнейшем оно понадобится при установке GitLab.

Далее требуется установить GitLab. Для этого в столбце "Подключиться" нажимаем кнопку "SSH". Во всплывающем окне откроется SSH-консоль созданной виртуальной машины. Последующие команды будем выполнять в ней.


Установим необходимые зависимости:
```bash
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates
```

Установим и сконфигурируем почтовый сервер (при настройке укажем отправку локальных уведомлений):
```bash
sudo apt-get install -y postfix   
```

Добавим репозиторий для установки пакетов GitLab:
```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash 
```

Установим GitLab Community Edition (вместо <gitlab-instance-ip-address> указываем внешний IP-адрес созданной виртуальной машины):
```bash
sudo EXTERNAL_URL="https://<gitlab-instance-ip-address>.xip.io" apt-get install gitlab-ce
```

После установки открываем внешний IP-адрес созданной виртуальной машины с дополнением ".xip.io" в бруаузере (протокол HTTPS, можно скопировать из вывода установщика GitLab) и попадаем в Web-интерфейс GitLab. Задаём пароль для пользователя root (например !QAZ2wsx). Всё - GitLab готов к работе. Окно с SSH консолью можно закрыть. Переходим к созданию репозитория с исходным кодом нашего приложения.

## Topic 2: Create repository with test application

Создадим группу для хранения набора репозиториев, которые будут использоваться в процессе курса. Нажимаем на кнопку "Cereate a group", в поле "Group name" вставляем имя группы "cicd", в поле "Group URL" вставляем префикс адреса "cicd", в поле "Group description (optional)" вставляем описание "Group for CICD repositories" и нажимаем кнопку "Create group".

Создадим репозиторий с кодом тестового проекта. Нажимаем на кнопку "New project", переходим на вкладку "Import project", нажимаем на кнопку "Repo by URL", в поле "Git repository URL" вставляем ссылку на тестовый репозиторий (https://github.com/jenoOvchi/hackeru-devops-go-simple.git) в поле "Project name" вписываем имя проекта "Test Project", в поле "Project description (optional)" вписываем описание проекта "Project with Go simple application" и нажимаем на кнопку "Create project".

## Topic 3: Build and run test application

Создадим пользователя для доступа к репозиторию исходного кода. Для этого нажимаем на кнопку "Admin Area", для создания пользователя нажимаем на кнопку "New User", в поле "Name" вписываем полное имя нового пользователя "Charles Chaplin", в поле "Username" вписываем имя нового пользователя в рамках Gitlab "cchaplin", в поле "Email" вписываем вымышленный почтовый ящик нового пользователя "cchaplin@gitlab.local" и нажимаем на кнопку "Create user".

При корректной настройке eMail и указании реальной почты на почтовый ящик пришло бы письмо с о регистрации в GitLab и ссылкой на обновление пароля. В нашем случае требуется задать пароль вручную. Для этого нажимаем на кнопку "Edit" и в полях "Password" и "Password confirmation" вписываем временный пароль "Q!W@e3r4" и нажимаем на кнопку "Save changes". После этого копируем ссылку на GitLab и открываем её в другом браузере/вкладке в режиме инкогнито. Вводим логин "cchaplin" и пароль "Q!W@e3r4" и переходим на форму смены временного пароля. Вводим в поле "Current password" текущий пароль "Q!W@e3r4", в поля "New password" и "Password confirmation" вводим новый пароль "!QAZ2wsx" и нажимаем на кнопку "Set new password". После этого мы оказываемся на странице нового пользователя в GitLab.

В данный момент новому пользователю не даны права на работу с созданной нами группой репозиториев. Добавим его в группу "cicd". Для этого вернёмся на вкладку GitLab под пользователем "root", нажмём на кнопку "Groups" и выберем пункт "Your groups". В списке групп выберем "cicd", перейдём на вкладку "Members", в поле "GitLab member or Email address" выберем пользователя "Charles Chaplin", в поле выбора роли пользователя "Choose a role permission" выберем "Maintainer" и нажмём кнопку "Invite". Вернёмся на вкладку GitLab под пользователем "cchaplin" и обновим страницу. В списке проектов появилась группа "cicd" и проект "Test Project".

Доступ к тестовому приложению появился, можем перейти к его сборке и запуску. Для этого создадим отдельную виртуальную машину, установим на неё Go, клонируем репозиторий с нашим приложением и осуществим его сборку и запуск.

Перейдём в Web интерфейс Google Cloud Platform и выберем в разделе Ресурсов вкладку Compute Engine. Необходимо создать новый экземпляр для сборки и запуска тестового приложения. Для этого нажмём кнопку "Создать экземпляр". Требуется задать имя виртуальной машины (test-app) и зону её размещения (us-central1-c). Для тестового приложения требуется минимум ресурсов, поэтому выбираем тип VM "g1-small". Необходимо будет выбрать также ОС. В разделе "Загрузочный диск" нажимаем кнопку "Изменить", выбираем Ubuntu 16.04 LTS и нажимаем кнопку "Выбрать". После этого в разделе "Брандмауэр" ставим флаг "Разрешить трафик HTTP" и нажимаем кнопку "Создать". Всё, ВМ создана. Обычно её развертывание занимает от 30 секунд до 1 минуты. После создания виртуальной машины обращаем внимание на значение в столбце "Внешний IP-адрес" - в дальнейшем оно понадобится при доступа к тестовому приложению.

Далее требуется клонировать репозиторий тестового приложения, осуществить его сборку и запуск. Для этого в столбце "Подключиться" нажимаем кнопку "SSH". Во всплывающем окне откроется SSH-консоль созданной виртуальной машины. Последующие команды будем выполнять в ней.

Установим Git (как правило установлен по умолчанию):
```bash
sudo apt-get install git
```

Клонируем репозиторий тестового приложения. Для этого в Web интерфейсе GitLab перейдём в проект "Test Project", нажмём на кнопку "Clone" и скопируем ссылку под надписью "Clone with HTTP". После этого выполним следующую команду, указав логин и пароль созданного пользователя (cchaplin/!QAZ2wsx):
```bash
git clone <git-repo-url>
```

Для сборки приложения установим Go:
```bash
sudo apt-get install -y wget
wget https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz
tar -xzf go1.13.3.linux-amd64.tar.gz
sudo mv go /usr/local
mkdir Projects
export GOROOT=/usr/local/go
export GOPATH=$HOME/Projects
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```

Проверим корректность установки:
```bash
go version
go env
```

Перейдём в директорию с тестовым приложением и соберём его:
```bash
cd test-project
go build -o hello
```

Запустим тестовое приложение:
```bash
nohup ~/test-project/hello > hello.log 2>&1 &
```

Проверим доступность тестового приложения:
```bash
curl localhost:3000
```

Скорректируем код тестового приложения для запуска на порту 80. Для этого перейдём в интерфейс GitLab, нажмём на файл "hello.go", нажмём на кнопку "Edit" в коде приложения заменим "3000" на "80", в поле "Commit message" напишем сообщение коммита "Change server port to 80" и нажмём на кнопку "Commit changes". На виртуальной машине скачаем актуальную версию кода из репозитория Git и пересоберём обновлённое приложение (cchaplin/!QAZ2wsx):
```bash
git pull
go build -o hello
```

Остановим старую версию приложения и запустим новую версию:
```bash
kill -9 $(ps -ef | grep hello | grep -v grep | awk '{print $2}')
sudo nohup ~/test-project/hello > hello.log 2>&1 &
```

Проверим доступность тестового приложения:
```bash
curl localhost
```

Проверим доступность тестового приложения в браузере, открыв внешний IP-адрес созданной виртуальной машины.

#### Задание:
- Создать репозиторий в GitLab на базе репозитория https://github.com/lamw/demo-go-webapp
- Создать виртуальную машину для выполнения заданий;
- Клонировать созданный репозиторий GitLab на новую виртуальную машину;
- Собрать тестовое приложение;
- Запустить собранное приложение и проверить его доступность по внешнему IP адресу виртуальной машины.

## Topic 3: Run application tests

Созданный нами репозиторий содержит набор тестов, проверяющих корректность работы тестового приложения. Запустим данный набор тестов и проверим, что они успешно выполняются:
```bash
go test
```

В процессе разработки очень важным является скорейшее получение обратной связи о качестве разрабатываемого кода. Для этого используются непрерывная интеграция и тестирование. Создадим для нашего приложения пайплайн с помощью GitLabCI, запускающий набор модульных тестов при появлении новых коммитов. Для этого в корневой директории репозитория нажмём кнопку "+" и выберем "New file". На открывшейся форме добавления нового файла вставим имя файла ".gitlab-ci.yml", нажмём "Select a template type", выберем ".gitlab-ci.yaml" и вставим следующий код:
```yaml
test:
  script: go test
```

В поле "Commit message" введём "Add .gitlab-ci.yml file" и нажмём "Commit changes". В левом меню нажмём на вкладку "CI / CD" и выберем пункт "Pipelines". Перейдём в экземпляр запущенног опайплайна и обнаружим, что у нас нет установленныйх раннеров для запуска данного пайплайна. Перейдём по ссылке "Runners page", на которой находится инструкция по установке раннера GitLab CI. Создадим для раннера отдельную виртуальную машину. 

Перейдём в Web интерфейс Google Cloud Platform и выберем в разделе Ресурсов вкладку Compute Engine. Необходимо создать новый экземпляр для настройки его в качестве GitLab Runner. Для этого нажмём кнопку "Создать экземпляр". Требуется задать имя виртуальной машины (gitlab-runner) и зону её размещения (us-central1-c). Для Runner'а требуется мало ресурсов, поэтому выбираем тип VM "g1-small". Необходимо будет выбрать также ОС. В разделе "Загрузочный диск" нажимаем кнопку "Изменить", выбираем Ubuntu 16.04 LTS и нажимаем кнопку "Выбрать". После этого нажимаем кнопку "Создать". Всё, ВМ создана. Обычно её развертывание занимает от 30 секунд до 1 минуты.

Далее требуется установить gitlab-runner и Go на создданной виртуальной машине для подключения её в качестве виртуальной машины для запуска пайплайнов GitLab и обеспечения возможности работы с кодом Golang. Установим gitlab-runner:
```bash
curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
sudo dpkg -i gitlab-runner_amd64.deb
sudo chmod +x /usr/bin/gitlab-runner
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
```

Установим Go для пользователя gitlab-runner:
```bash
sudo apt-get install -y wget
wget https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz
tar -xzf go1.13.3.linux-amd64.tar.gz
sudo mv go /usr/local
sudo mkdir /home/gitlab-runner/Projects
sudo chown gitlab-runner /home/gitlab-runner/Projects
sudo vi /home/gitlab-runner/.profile
```

```ini
...
GOROOT=/usr/local/go
GOPATH=$HOME/Projects
PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```

Зарегистрируем установленный gitlab-runner на сервере GitLab:
```bash
sudo gitlab-runner register
```

В качестве "gitlab-ci coordinator URL" укажем адрес из пункта 2 раздела "Set up a specific Runner manually" на странице "CI / CD Settings" в GitLab. В качестве "gitlab-ci token" укажем токен из пункта 3 раздела "Set up a specific Runner manually" на странице "CI / CD Settings" в GitLab. В качестве описания Runner'а укажем "Runner for Go apps". При задании "gitlab-ci tags" не будем указывать какие-либо теги для упрощения кода пайплайна. При задании "executor" укажем "shell".

Запустим gitlab-runner:
```bash
sudo gitlab-runner start
```

Проверим, что пайплайн выполнился. Переходим в раздел "CI / CD/Pipelines" и выбираем созданный запуск пайплайна. Нажимаем на шаг "test" и изучаем логи пайплайна. В них видно, что тестирование завершилось неуспешно из-за отсутствия необходимых библиотек компилятора. Чтобы избежать этой ошибки воспользуемся настройкой переменных пайплайна. Перейдём в раздел "Settings/CI / CD", откроем раздел "Variables" и добавим переменную с ключом "CGO_ENABLED", значением "0" и типом "Var". Перейдём в раздел "CI / CD/Pipelines", нажмём кнопку "Run Pipeline", выберем созданный запуск пайплайна, нажмём на шаг "test" и изучаем логи пайплайна. Как видно, пайплайн успешно отработал.

После того как мы убедились, что приложение удовлетворяет предъявляемым к нему функциональным требованиям, требуется обеспечить сборку новой версии приложения для её последующего распространения. Все, что для этого нужно сделать — определить еще одну задачу для CI. Назовем ее "package". Для этого перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна:
```yaml
test:
  script: go test

package:
  script: go build -o hello-app
```

В поле "Commit message" напишем сообщение коммита "Add package step" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Нажимаем на шаг "package" и изучаем логи пайплайна. Приложение было успешно собрано, но никуда не сохранено. Добавим сохранение артефакта после сборки. Для этого перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна:
```yaml
test:
  script: go test

package:
  script: go build -o hello-app
  artifacts:
    paths:
    - hello-app
```

В поле "Commit message" напишем сообщение коммита "Add package capturing" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Нажимаем на шаг "package" и изучаем логи пайплайна. В разделе "Job artifacts" нажмём кнопку "Browse" и увидим сохранённый нами артефакт. Нажмём на него и ссылка на его скачивание станет доступной. Скачаем его. В данный момент шаги пайплайна выполняются параллельно. Нам же требуется сохранять только те артефакты, которые упешно прошли тестирование. Добавим этапность в созданный нами пайплайн. Для этого перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна:
```yaml
stages:
  - test
  - package

test:
  stage: test
  script: go test

package:
  stage: package
  script: go build -o hello-app
  artifacts:
    paths:
    - hello-app
```

В поле "Commit message" напишем сообщение коммита "Add stages" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Видим, что у пайплайна появились этапы, выполняемые последовательно. Помимо этапов сборки и тестирования в пайплайны CI/CD добавляется развёртывание новых версий приложения на тестовые и продуктивную среды. Для обеспечения возможности развёртывания обновлений на виртуальной машине "test-app" с помощью GitLab CI настроим SSH доступ с виртуальной машины "gitlab-runner" на виртуальную машину "test-app" под пользователем "gitlab-runner". Первым делов перейдём на виртуальную машину "test-app" и создадим пользователя, под которым будем ходить по SSH с виртуальной машины "gitlab-runner" на виртуальную машину "test-app":
```bash
sudo useradd testapp -d /home/testapp -m
```

Добавим данному пользователю пароль, необходимый для первоначальной настройки SSH (например, !QAZ2wsx):
```bash
sudo passwd testapp
```

Добавим пользователя "testapp" в список sudo для безпарольного запуска приложения на порту 80:
```bash
sudo usermod -aG sudo testapp
echo "testapp ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/dont-prompt-testapp-for-password
```

Активируем возможность подключения по SSH по паролю:
```bash
sudo sed -i -r 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo service sshd restart
```

На виртуальной машине "gitlab-runner" создадим SSH ключ для пользователя "gitlab-runner":
```bash
sudo -u gitlab-runner ssh-keygen
```

Перенесём SSH ключ пользователя "gitlab-runner" на виртуальную машину тестового приложения (подставить в <test-app-ip> IP адрес виртуальной машины "test-app", пароль !QAZ2wsx):
```bash
sudo su gitlab-runner
ssh-copy-id -i /home/gitlab-runner/.ssh/id_rsa testapp@<test-app-ip>
```

Добавим 

Добавим в созданный нами пайплайн этап развёртывания обновлённой версии приложения на виртуальной машине "test-app". Для этого перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна (подставить в <test-app-ip> IP адрес виртуальной машины "test-app"): 
```yaml
stages:
  - test
  - package
  - deploy

test:
  stage: test
  script:
  - go clean
  - go test

package:
  stage: package
  script: go build -o hello-app
  artifacts:
    paths:
    - hello-app

deploy:
  stage: deploy
  script:
  - ssh testapp@<test-app-ip> "set +e && sudo kill -9 \$(ps -ef | grep hello | grep -v nohup | grep -v grep | awk '{print \$2}') && set -e"
  - scp hello-app testapp@<test-app-ip>:~/
  - ssh testapp@<test-app-ip> "chmod +x hello-app"
  - ssh testapp@<test-app-ip> "sudo nohup /home/testapp/hello-app > hello.log 2>&1 &"
```

В поле "Commit message" напишем сообщение коммита "Add deploy stage" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Видим, что старая версия приложения была остановлена, обновление было перенесено на виртуальную машину "test-app" и новая версия приложения была запущена. Продемонстрируем работоспособность пайплайна более наглядно - изменим текст приложения, возвращаемый Web сервером. Для этого перейдём в раздел "Repository", выберем файл "hello.go", нажмём на кнопку "Edit" и скорректируем строку №9:
```go
...
    fmt.Fprint(res, "Hello For All")
...
```

В поле "Commit message" напишем сообщение коммита "Change response" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Видим, что пайплайн упал на этапе запуска тестов. Возвращаемый текст изменился, в связи с чем тесты перестали проходить. Это хороший пример демонстрации контроля качества в рамках процесса "CI/CD" - обновление, не прошедшее контроль качества, не было доставлено на среду эксплуатации приложения. Обновим тесты и проверим, что приложение успешно обновлено. Для этого перейдём в раздел "Repository", выберем файл "hello_test.go", нажмём на кнопку "Edit" и скорректируем строку №18:
```go
...
    exp := "Hello For All"
...
```

В поле "Commit message" напишем сообщение коммита "Fix tests" и нажмём на кнопку "Commit changes". Перейдём в раздел "CI / CD/Pipelines" и выберем созданный запуск пайплайна. Видим, что пайплайн успешно отработал. Проверим, что приложение обновилось. Для этого откроем внешний IP-адрес виртуальной машины "test-app" и увидим, что выводимый текст обновился на "Hello For All".

#### Задание:
Для репозитория в GitLab на базе репозитория https://github.com/lamw/demo-go-webapp:
- Создать виртуальную машину для нового раннера;
- Установить раннер и добавить его к репозиторию;
- Настроить на виртуальной машине приложения доступ по SSH с раннера;
- Создать пайплайн для доставки тестового приложения;
- Внести в код изменение и проверить, что новая версия приложения успешно доставлена.

## Topic 4: Move test application to Docker

Первое, что требуется сделать с приложением для его успешной эксплуатации в Kubernetes - это его контейнеризация. Наиболее популярным средством контейнеризации в данный момент является Docker. Для начала остановим текущую версию запущенного приложения и установим Docker для дальнейшего запуска приложения в контейнере. Перейдём на виртуальную машину "test-app" и произведём установку:
```bash
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

Проверим работоспособность Docker:
```bash
sudo docker run hello-world
```

Для контейнеризации приложения требуется создать Dockerfile, описывающий процесс создания образа приложения на декларативном языке Docker. Для его создания перейдём в раздел "Repository", нажмём кнопку "+", выберем файл "New file", в списке "Select a template type" выберем "Dockerfile" и в списке "Apply a template" выберем "Golang-alpine". В поле "Commit message" напишем "Add Dockerfile" и нажмём кнопку "Commit changes".

На виртуальной машине "test-app" перейдём в папку с тестовым приложением и синхронизируем файлы с веткой "master" репозитория в Gitlab (cchaplin/!QAZ2wsx):
```bash
cd test-project/
git pull
```

Создадим группу "docker" и добавим текущего пользователя в эту группу для запуска Docker без sudo:
```bash
sudo groupadd docker
whoami | xargs sudo usermod -aG docker
```

Заново залогинимся на виртуальной машине, пепезагрузим Docker и проверим его доступность:
```bash
sudo systemctl restart docker
docker run hello-world
```

Соберём образ Docker с тестовым приложением:
```bash
cd test-project/
docker build -t helloworld-app:v1 .
```

Проверим созданный образ:
```bash
docker images
```

Запустим контейнер Docker с тестовым приложением:
```bash
docker run --name helloworld-app-container -p 8080:80 -d helloworld-app:v1
```

Изучим запущенный контейнеры:
```bash
docker ps
```

Проверим работу тестового приложения в контейнере:
```bash
curl localhost:8080
docker inspect helloworld-app-container
docker exec -it helloworld-app-container sh
ls /
ps aux
exit
ps aux | grep ./app | grep -v grep
```

Останавливаем запущенный контейнер:
```bash
docker stop helloworld-app-container
```

Удаляем остановленный контейнер:
```bash
docker rm helloworld-app-container
```

Запустим реестр образов Docker локально:
```bash
docker run -d -p 5000:5000 --name registry registry:2
```

Тегируем образ именем созданного реестра и загружаем его в реестр образов:
```bash
docker image tag helloworld-app:v1 localhost:5000/helloworld-app:v1
docker push localhost:5000/helloworld-app:v1
```

Удаляем локальный тег образа и заново скачиваем его из реестра образов:
```bash
docker rmi localhost:5000/helloworld-app:v1
docker pull localhost:5000/helloworld-app:v1
```

Запускаем контейнер из скачанного образа и проверяем его доступность:
```bash
docker run --name helloworld-app-container -p 8080:80 -d localhost:5000/helloworld-app:v1
curl http://localhost:8080
```

Останавливаем и удаляем запущенные контейнеры:
```bash
docker container stop helloworld-app-container && docker container rm -v helloworld-app-container
docker container stop registry && docker container rm -v registry
```

В GitLab интегрирован реестр образов Docker. Настроим к нему доступ. Для этого сначала протегируем виртуальную машину "gitlab" соответствующим тегом. Перейдём в Web интерфейс Google Cloud Platform и выберем в разделе Ресурсов вкладку Compute Engine. Необходимо открыть экземпляр "gitlab" и нажать кнопку "изменить". В разделе "Теги сети" добавляем тег "registry", нажимаем "Enter" и кнопку сохранить. После этого в разделе Ресурсов перейдём на вкладку "Сеть VPC/Правила брандмауэра" и нажмём кнопку "создать правило брандмауэра". В поле название введём "registry", в поле "Описание" введём "Open port for GitLab docker registry", в поле "Теги целевых экземпляров" введём "registry", в поле "Диапазон IP-адресов источников" введём "0.0.0.0/0", а в списке "Указанные протоколы и порты" выберем "tcp", укажем порт "5050" и нажмём кнопку "создать". Доступ к реестру образов Docker в GitLab открыт, приступаем к его использованию.

Возвращаемся на виртуальную машину "test-app" и авторизуемся в реестре образов Docker нашего GitLab. Для этого открываем Web интерфейс GitLab и переходим в разде "Packages/Container Registry" репозитория. В разделе "Quick Start" копируем команду авторизации и авторизуемся в реестре (cchaplin/!QAZ2wsx):
```bash
docker login <gitlab-instance-ip-address>.xip.io:5050
```

Тегируем образ нашего приложения адресом реестра образов GitLab и загружаем его в реестр:
```bash
docker tag helloworld-app:v1 <gitlab-instance-ip-address>.xip.io:5050/cicd/test-project:v1
docker push <gitlab-instance-ip-address>.xip.io:5050/cicd/test-project:v1
```

Проверим, что образ успешно загружен в реестр. Для этого переходим в раздел "Packages/Container Registry" и видим, что там появился образ с именем "cicd/test-project". Нажимаем на него и видим, что там есть образ с тегом "v1". Пробуем загрузить его на виртуальной машине "test-app". Для этого удаляем локальный тег образа и заново скачиваем его из реестра образов:
```bash
docker rmi <gitlab-instance-ip-address>.xip.io:5050/cicd/test-project:v1
docker pull <gitlab-instance-ip-address>.xip.io:5050/cicd/test-project:v1
```

Теперь, когда наше приложение докеризовано, настроим доставку обновлений приложения в виде образов Docker и их запуск в контейнерах. Для этого сначала перейдём на виртуальную машину "test-app" и остановим приложение, запущенное на узле через "nohup":
```bash
sudo kill -9 $(ps -ef | grep hello-app | grep -v nohup | grep -v grep | awk '{print $2}')
```

Далее установим на виртуальную машину "gitlab-runner" Docker:
```bash
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

Проверим работоспособность Docker:
```bash
sudo docker run hello-world
```

Создадим группу "docker" и добавим текущего пользователя в эту группу для запуска Docker без sudo:
```bash
sudo groupadd docker
whoami | xargs sudo usermod -aG docker
```

Заново залогинимся на виртуальной машине, пепезагрузим Docker и проверим его доступность:
```bash
sudo systemctl restart docker
docker run hello-world
```

Добавим пользователя "gitlab-runner" в группу "docker":
```bash
sudo usermod -aG docker gitlab-runner
```

Проверим, что у пользователя "gitlab-runner" доступно использование Docker:
```bash
sudo -u gitlab-runner -H docker info
```

В созданном нами пайплайне изменим этап "package", указав в нём команды для сборки образа Docker и его загрузки во внутренний реестр GitLab. Для этого перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна:
```yaml
stages:
  - test
  - package

test:
  stage: test
  script:
  - go clean
  - go test

package:
  stage: package
  before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
  - docker build -t $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID .
  - docker push $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID
```

Переходим в созданный экземпляр пайплайна и проверяем, что все шаги выполнены успешно. Проверим, что образ тестового микросервиса успешно сохранён во внутренний реестр образов Docker GitLab. Для этого переходим в раздел "Packages/Container Registry", нажимаем на репозиторий "cicd/test-project" и видим, что там есть образ с тегом, аналогичным номеру запуска пайплайна.

Добавим в пайплайн этап развёртывания приложения из созданного образа Docker. Для этого на виртуальной машине "test-app" добавим пользователя "testapp" в группу "docker":
```bash
sudo usermod -aG docker gitlab-runner
```

Далее в Web интерфейсе GitLab перейдём в раздел "Repository", выберем файл ".gitlab-ci.yml", нажмём на кнопку "Edit" и скорректируем описание шагов пайплайна (подставить в <test-app-ip> IP адрес виртуальной машины "test-app"):
```yaml
stages:
  - test
  - package
  - deploy

test:
  stage: test
  script:
  - go clean
  - go test

package:
  stage: package
  before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
  - docker build -t $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID .
  - docker push $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID

deploy:
  stage: deploy
  before_script:
  - ssh testapp@<test-app-ip> "docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY"
  script:
  - ssh testapp@<test-app-ip> "docker pull $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID"
  - ssh testapp@<test-app-ip> "docker stop helloworld-app-container || true && docker rm helloworld-app-container || true"
  - ssh testapp@<test-app-ip> "docker run --name helloworld-app-container -p 80:80 -d $CI_REGISTRY/cicd/test-project:$CI_PIPELINE_IID"
```

#### Задание:
Для репозитория в GitLab на базе репозитория https://github.com/lamw/demo-go-webapp:
- Установить Docker на виртуальной машине со старой версией приложения;
- Добавить Dockerfile в репозиторий приложения;
- Загрузить изменения из репозитория на машину со старой версией приложения, остановить старую версию, собрать образ Docker с приложением, запустить контейнер и проверить доступность приложения;
- Скорректировать пайплайн приложения для обеспечения его доставки с помощью Docker;
- Внести в код изменение и проверить, что новая версия приложения успешно доставлена.

## Topic 5: Use Docker Compose

Современные приложения, особенно разрабатываемые в микросервисной архитектуре, чаще всего состоят из набора контейнеров, которые разворачиваются совместно. Для этих целей может использоваться инструмент Docker Compose, позволяющий описать группу контейнеров, запускаемых и управляемых совместно и единообразно. Для демонстрации его работы создадим ещё один репозиторий и импортируем в него приложение, состоящее из двух контейнеров - Web приложения и БД Redis, которая используется для кеширования данных, запрашиваемых пользователями.

В Web интерфейсе GitLab перейдём в группу "cicd" и нажмём на кнопку "New Project", переходим на вкладку "Import project", нажимаем на кнопку "Repo by URL", в поле "Git repository URL" вставляем ссылку на тестовый репозиторий (https://github.com/callicoder/go-docker-compose.git) в поле "Project name" вписываем имя проекта "Compose Test", в поле "Project description (optional)" вписываем описание проекта "Show Docker Compose in action" и нажимаем на кнопку "Create project".

Продемонстрируем работу Docker Compose. Для этого перейдём на виртуальную машину "test-app", остановим и удалим запущенный контейнер "helloworld-app-container":
```bash
docker stop helloworld-app-container
docker rm helloworld-app-container
```

Клонируем репозиторий тестового приложения. Для этого в Web интерфейсе GitLab перейдём в проект "Compose Test", нажмём на кнопку "Clone" и скопируем ссылку под надписью "Clone with HTTP". После этого выполним следующую команду, указав логин и пароль созданного пользователя (cchaplin/!QAZ2wsx):
```bash
git clone <git-repo-url>
```

Для запуска приложения с помощью Docker Compose для начала установим Docker Compose:
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Проверим работоспособность Docker Compose:
```bash
docker-compose --version
```

Перейдём в папку с клонированным репозиторием и изучим файл Docker Compose:
```bash
cd compose-test/
cat docker-compose.yml
```

Соберём и запустим приложение с помощью Docker Compose:
```bash
docker-compose up
```

При этом логи обоих контейнеров (приложения и Redis) будут выведены в стандартный вывод терминала. Для их запуска в фоновом режиме завершим текущий процесс и запустим этот же набор контейнеров в фоновом режиме:
```bash
docker-compose up -d
```

Изучим статус запущенных контейнеров:
```bash
docker-compose ps
```

Протестируем работу приложения на локальном хосте:
```bash
curl http://localhost:8080
```

Для обеспечения доступа к приложению из браузера вернёмся в Web интерфейсе GitLab, перейдём в проект "Compose Test" и внесём изменения в файл docker-compose.yaml:
```yaml
# Docker Compose file Reference (https://docs.docker.com/compose/compose-file/)

version: '3'

# Define services
services:

  # App Service
  app:
    # Configuration for building the docker image for the service
    build:
      context: . # Use an image built from the specified dockerfile in the current directory.
      dockerfile: Dockerfile
    ports:
      - "80:8080" # Forward the exposed port 8080 on the container to port 8080 on the host machine
    restart: unless-stopped
    depends_on: 
      - redis # This service depends on redis. Start that first.
    environment: # Pass environment variables to the service
      REDIS_URL: redis:6379    
    networks: # Networks to join (Services on the same network can communicate with each other using their name)
      - backend

  # Redis Service   
  redis:
    image: "redis:alpine" # Use a public Redis image to build the redis service    
    restart: unless-stopped
    networks:
      - backend

networks:
  backend:
```

Остановим и удалим запущенный набор контейнеров и скачаем изменения docker-compose.yml:
```bash
docker-compose stop
docker-compose rm
git pull
```


Запустим приложение в фоновом режиме и проверим его доступность с хоста:
```bash
docker-compose up -d
curl http://localhost/qod
```

Откроем в браузере внешний IP адрес виртуальной машины "test-app" и проверим доступность приложения извне.

Теперь настроим для данного приложения процесс непрерывной доставки. Для этого добавим соответствующий файл ".gitlab-ci.yml" в репозиторий с его кодом. Для этого перейдём в Web интерфейс Gitlab и в корневой директории репозитория "Compose Test" нажмём кнопку "+" и выберем "New file". На открывшейся форме добавления нового файла вставим имя файла ".gitlab-ci.yml", нажмём "Select a template type", выберем ".gitlab-ci.yaml" и вставим следующий код:
```yaml
stages:
  - package
  - deploy

package:
  stage: package
  before_script:
  - ssh testapp@<test-app-ip> "rm -rf /home/testapp/compose-test"
  script:
  - scp -r "${PWD}" testapp@<test-app-ip>:/home/testapp
  - ssh testapp@<test-app-ip> "docker-compose -f compose-test/docker-compose.yml build"
  only:
    - master

deploy:
  stage: deploy
  script:
  - ssh testapp@<test-app-ip> "docker-compose -f compose-test/docker-compose.yml up -d"
  only:
    - master
```

После этого в поле "Commit message" введём "Add .gitlab-ci.yml" и нажмём кнопку "Commit changes". Заметим, что указан тег "only" со значением "master", благодаря которому данный пайплайн будет выполняться только при появлении коммитов в ветке "master". Запустится пайплайн, но его первая задача будет висеть в статусе "pending" поскольку у данного проекта нет раннеров. Добавим существующий раннер к проекту "Compose Test". Для этого откроем Web интерфейс Gitlab, авторизованный под пользователем "root", нажмём на кнопку "Admin Area", перейдём в раздел "Overview/Runners", выберем подключенный раннер и в подразделе "Restrict projects for this Runner" нажмём кнопку "Enable" в строке "cicd / Compose Test". После этого данный раннер становится доступным проекту "Compose Test" для запуска пайплайнов. Вернёмся в Web интерфейс Gitlab под пользователем "Charles Chaplin" и проверим, что пайплайн успешно выполнился.

Откроем в браузере внешний IP адрес виртуальной машины "test-app" и проверим доступность приложения извне.

Теперь, по аналогии с примером без использования Docker Compose, настроим сборку и публикацию образов Docker во внутренний Registry GitLab и их использование при развёртывании. Для этого для начала перейдём на виртуальную машину "test-app", на которой остановим и удалим текущий экземпляр:
```bash
sudo docker-compose -f /home/testapp/compose-test/docker-compose.yml stop
sudo docker-compose -f /home/testapp/compose-test/docker-compose.yml rm -f
sudo rm -rf /home/testapp/compose-test
```

Создадим отдельную ветку в репозитории "Compose Test", в рамках которой будем осуществлять доработку пайплайна и файла Docker Compose. Вернёмся в Web интерфейс Gitlab под пользователем "Charles Chaplin", перейдём в проект "Compose Test", нажмём на кнопку "+" и выберем "New branch". На открывшейся форме в поле "Branch name" введём "issue-1" и нажмём кнопку "Create branch". Будет осуществлён переход на созданную ветку.

Для начала доработаем файл "docker-compose.yml". Выберем его в списке файлов, нажмём на кнопку "Edit" и изменим его содержимое на следующее:
```yaml
version: '3'

# Define services
services:

  # App Service
  app:
    # Configuration for building the docker image for the service
    image: "${REGISTRY}/${GROUP}/${REPO}:${TAG}"
    ports:
      - "80:8080" # Forward the exposed port 8080 on the container to port 8080 on the host machine
    restart: unless-stopped
    depends_on: 
      - redis # This service depends on redis. Start that first.
    environment: # Pass environment variables to the service
      REDIS_URL: redis:6379    
    networks: # Networks to join (Services on the same network can communicate with each other using their name)
      - backend

  # Redis Service   
  redis:
    image: "redis:alpine" # Use a public Redis image to build the redis service    
    restart: unless-stopped
    networks:
      - backend

networks:
  backend:
```

После этого в поле "Commit message" введём "Change config from build image to use image from GitLab Registry." и нажмём кнопку "Commit changes". Обратим внимание, что пайплайн при этом не был запущен.

Теперь доработаем описание пайплайна. Вернёмся в корневую директорию репозитория, выберем файл ".gitlab-ci.yml", нажмём кнопку "Edit" и заменим содержимое файла на следующее: 
```yaml
stages:
  - package
  - deploy

package:
  stage: package
  before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
  - docker build -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME:$CI_PIPELINE_IID .
  - docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME:$CI_PIPELINE_IID
  only:
    - master

deploy:
  stage: deploy
  before_script:
  - ssh testapp@<test-app-ip> "docker-compose stop || true"
  - ssh testapp@<test-app-ip> "docker-compose rm -f || true"
  - ssh testapp@<test-app-ip> "rm docker-compose.yml || true"
  - ssh testapp@<test-app-ip> "docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY"
  script:
  - scp docker-compose.yml testapp@<test-app-ip>:~/docker-compose.yml
  - ssh testapp@<test-app-ip> "echo \"\" > .env; echo REGISTRY=$CI_REGISTRY >> .env; echo GROUP=$CI_PROJECT_NAMESPACE >> .env; echo REPO=$CI_PROJECT_NAME >> .env; echo TAG=$CI_PIPELINE_IID >> .env;"
  - ssh testapp@<test-app-ip> "docker pull $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME:$CI_PIPELINE_IID"
  - ssh testapp@<test-app-ip> "docker-compose up -d"
  only:
    - master
```

После этого в поле "Commit message" введём "Add save image to GitLab Registry and use Docker Compose with variables steps." и нажмём кнопку "Commit changes".

Теперь, для того чтобы перенести изменения в ветку "master" нажмём на кнопку "+", выберем пункт "New merge request", в списке "Select source branch" выберем "issue-1" и нажмём "Compare branches and continue". В поле "Title" "Merge issue 1 changes", в поле "Description" введём "Update pipeline and Docker Compose files", нажмём кнопку "Assign to me" и кнопку "Submit merge request". На открывшейся форме перейдём в подраздел "Commits" и изучим переносимые коммиты. Перейдём в подраздел "Changes" и изучим внесённые изменения. После этого вернёмся в подраздел "Overview" и нажмём кнопку "Merge". Перейдём в подраздел "CI/CD / Pipelines" и проверим, что пайплайн успешно выполнился.

Откроем в браузере внешний IP адрес виртуальной машины "test-app" и проверим доступность приложения извне.

Теперь для демонстрации непрерывной доставки внесём изменение в код и продемонстрируем доставку обновлений по коммиту. Перейдём в подраздел "Repository / Files", выберем файл "app.go" и нажмём кнопку "Edit". В строке 20 заменим текст "Welcome! Please hit the `/qod` API to get the quote of the day." на текст "Welcome to the new version of the app!", в поле "Commit message" введём "Change text of the indexHandler function." и нажмём кнопку "Commit changes". Перейдём в подраздел "CI/CD / Pipelines" и проверим, что пайплайн успешно выполнился.

Откроем в браузере внешний IP адрес виртуальной машины "test-app" и проверим, что текст стартовой страницы был изменён.

#### Задание:
Для репозитория в GitLab на базе репозитория https://github.com/lamw/demo-go-webapp:
- Установить Docker Compose на виртуальной машине со старой версией приложения - сделать композицию из приложения и Redis;
- Добавить файл docker-compose.yml в репозиторий приложения;
- Загрузить изменения из репозитория на машину со старой версией приложения, остановить старую версию, запустить новую версию с помощью Docker Compose и проверить доступность приложения;
- Скорректировать пайплайн приложения для обеспечения его доставки с помощью Docker Compose;
- Внести в код изменение и проверить, что новая версия приложения успешно доставлена.

