# arch-wsl

**📝 Итоговая инструкция по установке и настройке Arch Linux на WSL (актуально на апрель 2025)**  
*Для Windows 11, Intel Core i7 13700K, NVIDIA RTX 4090*

---

### **1. Подготовка Windows 11**  
#### **1.1 Включение компонентов**  
1. **Виртуализация**:  
   - В BIOS/UEFI активируйте:  
     - **Intel Virtualization Technology (VT-x)**.  
     - **Hyper-V** (если доступно).  
   - В Windows:  
     - `Win + R` → `optionalfeatures` → включите:  
       - *Hyper-V*,  
       - *Virtual Machine Platform*,  
       - *Windows Subsystem for Linux*.  

2. **Обновление WSL**:  
   ```powershell
   wsl --update
   wsl --set-default-version 2
   ```

---

### **2. Установка Arch Linux WSL**  
#### **2.1 Импорт образа**  
1. Скачайте актуальный `.wsl`-образ с [официального репозитория](https://gitlab.archlinux.org/archlinux/archlinux-wsl).  
2. Импортируйте дистрибутив:  
   ```powershell
   wsl --import ArchLinux C:\WSL\ArchLinux C:\Downloads\archlinux-latest.wsl --version 2
   ```

#### **2.2 Первичная настройка**  
```powershell
wsl -d ArchLinux
```  
После запуска выполните:  
```bash
/usr/lib/wsl/first-setup.sh  # Обязательный скрипт для systemd
```

---

### **3. Базовая настройка Arch Linux**  
#### **3.1 Включение Multilib репозитория**  
```bash
# Раскомментировать Multilib репозиторий
sed -i '/\[multilib\]/,/Include/ s/^#//' /etc/pacman.conf

# Обновление системы после включения репозитория
pacman -Syu --noconfirm
```

#### **3.2 Локализация и время**  
```bash
# Раскомментируем русскую и английскую локали
sed -i 's/#\(en_US\|ru_RU\)\.UTF-8 UTF-8/\1.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen
echo "LANG=ru_RU.UTF-8" > /etc/locale.conf  # Системный язык — русский

# Московское время
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
```

#### **3.3 Настройка WSL**  
Создайте файл `/etc/wsl.conf`:  
```bash
echo "
[user]
default=archlyze  # Пользователь по умолчанию

[interop]
enabled=true  # Взаимодействие с Windows
appendWindowsPath=true
" > /etc/wsl.conf
```

---

### **4. Создание пользователя**  
```bash
pacman -S --noconfirm zsh  # Устанавливаем zsh заранее
useradd -m -G wheel -s /bin/zsh archlyze  # Только группа wheel
passwd archlyze  # Установите пароль
echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/wheel  # Отключение пароля для sudo
```

---

### **5. Установка драйверов и компонентов NVIDIA**  
#### **5.1 Драйверы NVIDIA**  
- **На Windows**:  
  Установите последнюю версию драйвера для RTX 4090 с [официального сайта](https://www.nvidia.com/Download/index.aspx).  

- **В Arch Linux**:  
  ```bash
  pacman -S --noconfirm nvidia-utils lib32-nvidia-utils cuda
  ```  

#### **5.2 Настройка графики (WSLg)**  
В файл `%USERPROFILE%\.wslconfig` добавьте:  
```ini
[wsl2]
guiApplications = true
kernelCommandLine = cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1
memory=16GB  # Рекомендуется для RTX 4090
```

---

### **6. Установка ПО**  
#### **6.1 Системные утилиты**  
```bash
pacman -S --noconfirm \
    nano \
    python \
    python-pip \
    htop \
    curl \
    wget \
    unzip \
    git
```

#### **6.2 ZSH + Oh My ZSH**  
```bash
# Для root
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended

# Для archlyze
cp -r /root/.oh-my-zsh /home/archlyze/
chown -R archlyze:archlyze /home/archlyze/.oh-my-zsh
echo 'export ZSH="/home/archlyze/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git docker)
source $ZSH/oh-my-zsh.sh' >> /home/archlyze/.zshrc
```

#### **6.3 Paru (AUR-хелпер)**  
```bash
pacman -S --needed base-devel git
sudo -u archlyze git clone https://aur.archlinux.org/paru-bin.git /tmp/paru
cd /tmp/paru
sudo -u archlyze makepkg -si

# Оптимизированный конфиг
echo "
[options]
SkipReview
BottomUp
ProvidesLevel = 9999" > /etc/paru.conf
```

#### **6.4 Docker с поддержкой GPU**  
```bash
pacman -S --noconfirm docker docker-compose nvidia-container-toolkit
systemctl enable docker
mkdir -p /etc/docker
echo '{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia"
}' > /etc/docker/daemon.json
systemctl restart docker
```

---

### **7. Проверка системы**  
#### **7.1 NVIDIA и CUDA**  
```bash
nvidia-smi  # Должна отображаться RTX 4090
docker run --rm --gpus all nvidia/cuda:latest-base nvidia-smi
```

#### **7.2 Локализация и время**  
```bash
locale  # LANG=ru_RU.UTF-8
timedatectl | grep "Time zone"  # Europe/Moscow
```

#### **7.3 ZSH и Docker**  
```bash
wsl -d ArchLinux  # Автоматический вход под archlyze
docker ps  # Проверка работы Docker
```

---

### **8. Финализация**  
```bash
# Обновление системы
pacman -Syu --noconfirm

# Перезапуск WSL
wsl --terminate ArchLinux
wsl -d ArchLinux
```

---

### **❗ Важные примечания**  
1. **Драйверы NVIDIA**:  
   - Устанавливаются **только в Windows** (версия ≥551.23 для RTX 4090).  
   - В Arch Linux пакеты `nvidia-utils` и `cuda` нужны только для утилит и библиотек.  

2. **Группы пользователя**:  
   - Группы `audio`, `video`, `nvidia` **не требуются** — доступ к оборудованию управляется через Windows.  

3. **WSLg**:  
   - Для GUI-приложений убедитесь, что в Windows установлен [драйвер WSLg](https://learn.microsoft.com/en-us/windows/wsl/wslg).  

4. **Multilib**:  
   - Репозиторий Multilib необходим для установки 32-битных библиотек, включая `lib32-nvidia-utils`.

---

### **🔗 Актуальные ссылки**  
- [ArchWiki: Установка Arch на WSL](https://wiki.archlinux.org/title/Install_Arch_Linux_on_WSL)  
- [NVIDIA CUDA в WSL](https://docs.nvidia.com/cuda/wsl-user-guide/index.html)  
- [Документация Docker с NVIDIA](https://docs.docker.com/config/containers/resource_constraints/#gpu)  

**Для обновления пакетов (2025):**  
```bash
paru -Syu  # AUR + официальные репозитории
```
