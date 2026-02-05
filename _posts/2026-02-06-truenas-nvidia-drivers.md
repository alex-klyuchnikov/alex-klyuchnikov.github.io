---
layout: post
title:  "Пакуем драйвера для видеокарты в TrueNAS Scale Goldeye"
categories: [TrueNAS Scale, NVidia]
tags: [truenas scale, goldeye, nvidia, драйверы]
---

IXSystems в TrueNAS Scale Goldeye 25.10 обновили драйвера на видеокарты NVidia до версии 570.172.08. Это спровоцировало две проблемы. Первая – старые видеокарты перестали поддерживаться. Вторая проблема вытекает из того, что стали поддерживаться новые видеокарты, но не все. Решить эту проблему можно заменив драйвера NVidia в TrueNAS Scale.

# Диагностика {#diagnostic} 

После обновления и передёргивания галочки Install NVIDIA Drivers в настройках приложений TrueNAS Scale, нужно сходить в Shell и запустить команду `nvidia-smi`. Если команда возвращает что-то похожее на то, что на картинке, поздравляю – вас не коснулась эта проблема.
![Вывод команды `nvidia-smi` с работающей видеокартой](assets/img/truenas-drivers-shell.png)

Если же в результате вы получаете `No devices were found`, значит драйвера всё же не подключились. Дополнительно можно проверить вызвав `dmesg` и поискав в выводе следующие строчки:
```
nvAssertFailedNoLog: Assertion failed: 0 @ kernel_gsp.c:1443
_kgspBootGspRm: (the GPU is likely in a bad state and may need to be reset)
```
А так же если `lspci` не выдаёт модель вашей видеокарты, а выдаёт что-то невнятное (типа `NVidia VGA Compatible Controller`), значит нужно менять драйвера.

# Решение {#solution} 

Я собрал драйвера NVidia версии 580.105.08 для версии TrueNAS SCALE 25.10.1 и положил их в свой Telegram канал. Если это те драйвера, которые вам нужны, то скачивайте их [отсюда](TODO вставить ссылку на пост в телеге) и переходите к [шагу #4](#drivers-installation).

Если вам нужны драйвера постарше (для видеокарт 10 серии и младше), то можно посмотреть в [репозитории](https://github.com/zzzhouuu/truenas-nvidia-drivers). 

Если же вам нужны драйвера поновее, или версия TrueNAS Scale уже обновилась, то нужно собрать драйвера самостоятельно. Делается это следующим образом:

## 1. Подготовка {#preparation} 

Скачиваем Debian 12 (bookworm) [отсюда](https://mirror.accum.se/cdimage/archive/12.13.0/amd64/) и устанавливаем его. Возможно подойдёт и Debian 13, но я собирал на двенадцатом. Если используете виртуальную машину – выделите минимум 16 гигабайт оперативной памяти, иначе скрипты сборки будут жаловаться. Затем устанавливаем нужные пакеты и устанавливаем переменные окружения для своего текущего релиза TrueNAS Scale:
```bash
sudo apt update

sudo apt install build-essential debootstrap git python3-pip python3-venv squashfs-tools unzip libjson-perl rsync libarchive-tools

git clone -b TS-25.10.1 https://github.com/truenas/scale-build.git
cd scale-build
export TRUENAS_TRAIN="TrueNAS-SCALE-Goldeye"
export TRUENAS_VERSION="25.10.1"
export PATH=$PATH:/usr/sbin:/sbin
```

Не забудьте заменить `TRUENAS_TRAIN` и `TRUENAS_VERSION` на свои, если они у вас отличаются. Так же в команде `git clone` замените `TS-25.10.1` на свою версию TrueNAS Scale.

## 2. Редактирование функции установки драйверов {#edit-function} 

На этом этапе нам нужно отредактировать скрипт скачивания и установки драйверов. Выполняем `nano scale_build/extensions.py` и ищем функцию `download_nvidia_driver`. `CTRL + W` в `nano` поможет искать, но если вы делаете это из браузера, то `CTRL + W` закроет вкладку, так что придётся искать глазами. Как нашли – выбираем из следующих вариантов:

### a. Если нужно поставить драйвера поновее или постарше {#alternative-drivers} 
Список версий драйверов – [здесь](https://download.nvidia.com/XFree86/Linux-x86_64/). Выбираем по вкусу, я выбрал 580.105.08.

```python
    def download_nvidia_driver(self):
        version = "580.105.08"
        prefix = "https://us.download.nvidia.com/XFree86/Linux-x86_64"
        filename = f"NVIDIA-Linux-x86_64-{version}.run"
        result = f"{self.chroot}/{filename}"
        self.run([
            "wget", "-c", "-O", f"/{filename}", f"{prefix}/{version}/{filename}"
        ])
        os.chmod(result, 0o755)
        return result
```

### b. Если нужно поставить Grid драйвера, с дополнительными параметрами установки {#grid-drivers} 
Нужно будет заменить ещё и функцию `install_nvidia_driver`. Я оставил оригинальные комментарии автора кода, но думаю, тут всё понятно, где и что заменять.

```python
    def download_nvidia_driver(self):
        # i don't where i can find the grid driver
        prefix = "<put your grid download url without filename in here>"
        #useless
        #version = get_manifest()["extensions"]["nvidia"]["current"]
        filename = f"<filename of your grid driver>"
        result = f"{self.chroot}/{filename}"

        self.run(["wget", "-c", "-O", f"/{filename}", f"{prefix}/{filename}"])

        os.chmod(result, 0o755)
        return result

    def install_nvidia_driver(self, kernel_version):
        driver = self.download_nvidia_driver()
        # fix option
        self.run([f"/{os.path.basename(driver)}", "--silent", f"--kernel-name={kernel_version}"])

        os.unlink(driver)
```

## 3. Сборка {#make} 

Выполняем следующие команды:

```bash
make checkout
sudo make packages
sudo make update

mkdir -p ./tmpfile/rootfs
sudo mount ./tmp/update/rootfs.squashfs ./tmpfile/rootfs

ls -al ./tmpfile/rootfs/usr/share/truenas/sysext-extensions/nvidia.raw
sudo cp ./tmpfile/rootfs/usr/share/truenas/sysext-extensions/nvidia.raw ~/nvidia_580.105.08.raw

sudo umount ./tmpfile/rootfs
rmdir ./tmpfile/rootfs
```

На виртуалке у меня это заняло космическое количество времени. В оригинальном посте пишут, что заняло около часа, у меня же это заняло несколько часов. Сборка будет падать по своим причинам (особенно во время `make packages`), поэтому придётся перезапустить несколько раз. Не стоит пугаться, когда-нибудь она дойдёт до конца.

В результате этих команд у вас в домашней директории образуется файл `nvidia_580.105.08.raw`, если вы не меняли версию. Этот файл нужно перенести на сервер с TrueNAS Scale (по SMB или SSH, кому как нравится больше).

## 4. Установка драйверов {#drivers-installation} 

И наконец, нам нужно установить полученный файл.

1. Если галочка `Install NVidia Drivers` в настройках приложений установлена, то выполняем:

    ``` bash
    sudo systemd-sysext unmerge
    ```

2. Переводим датасет `/usr` в режим записи:

    ```bash
    sudo zfs set readonly=off “$(zfs list -H -o name /usr)”
    ```

3. Делаем резервную копию старых драйверов и копируем новые:

    ```bash
    sudo mv /usr/share/truenas/sysext-extensions/nvidia.raw /usr/share/truenas/sysext-extensions/nvidia.bak

    sudo cp ~/nvidia_580.105.08.raw /usr/share/truenas/sysext-extensions/nvidia.raw
    ```

4. Переводим датасет `/usr` в режим только-чтения:

    ```bash
    sudo zfs set readonly=on “$(zfs list -H -o name /usr)”
    ```

5. Мержим `systemd-sysext`:

    ```bash
    sudo systemd-sysext merge
    ```

6. Перезагружаемся.

После перезагрузки запускаем `nvidia-smi` и радуемся, если в выводе видим свою видеокарту.

Ссылки, использовавшиеся для вдохновения, с подробностями о том как оно всё работает:

[NVIDIA Kernel Module Change in TrueNAS 25.10 - What This Means for You #105](https://forums.truenas.com/t/nvidia-kernel-module-change-in-truenas-25-10-what-this-means-for-you/51070/105)

[TrueNAS Build Nvidia vGPU Driver extensions (systemd-sysext)](https://www.homelabproject.cc/posts/truenas/truenas-build-nvidia-vgpu-driver-extensions-systemd-sysext/)

[TrueNAS 25.10 Nvidia GPU Driver](https://github.com/zzzhouuu/truenas-nvidia-drivers/tree/main)

Ну и подписывайтесь на мой [Телеграм канал](https://t.me/klyuchnikov_channel), конечно же. А комментировать статью можно вот под [этим постом](https://t.me/klyuchnikov_channel/194).
