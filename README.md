# Jetson a Ubuntu kurduktan sonra izlenecek ilk adımlar
### Jetson ın IP adresini al ve ssh ile bağlan
MacOS'de "lanscan", Windows'da "wnetwatcher", ve Linux'de "angry ip scanner" kullanabilirsin. Veya terminal i kullanarak IP tarayabilirsin:
```bash
sudo apt-get update && sudo apt-get install arp-scan
sudo arp-scan --localnet ## Burada --interface seçeneğini araştırman gerekebilir

ssh nvidia@<buraya IP adresi>
# örnek:
ssh nvidia@192.168.1.110 # varsayılan şifre: nvidia
```

### Şifreyi değiştir
Şifreyi varsayılan olarak bırakmak iyi bir fikir değildir
```bash
passwd
```

### Hostname i değiştir
Her seferinde IP adresini aramak yerine yerel domain kullanmak için.
  Ascii Cinema sürümü: https://asciinema.org/a/nytXz7ZUMGAXb6VpHY0fLHEJY
```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install avahi-daemon

sudo nano /etc/hosts
# Hostname i istediğin gibi değiştir

sudo nano /etc/hostname
# /etc/hosts a yazdığın hostname in aynısını buraya yaz

sudo /etc/init.d/hostname.sh

sudo reboot
```
Şimdi hostname i kullanarak giriş yap:
```bash
ssh nvidia@<Buraya hostname>.local
# örnek:
ssh nvidia@teamName.local
```
### nano'yu kur (isteğe bağlı, şiddetle tavsiye edilir)
```bash
sudo apt-get install nano
```
### zsh'i kur (isteğe bağlı)
Ascii cinema: https://asciinema.org/a/x6QlETqxDvdK8t0lmBvOWVoKD

### screen'i kur
```bash
sudo apt-get install screen
```

## Scanse SDK sını kur
```bash
cd ~/
# clone the sweep-sdk repository
git clone https://github.com/scanse/sweep-sdk

# enter the libsweep directory
cd sweep-sdk/libsweep

# create and enter a build directory
mkdir -p build
cd build

# build and install the libsweep library
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
sudo cmake --build . --target install
sudo ldconfig
```
## Racecar ı kur


```bash
cd ~/
wget --no-parent -nH http://ec2-18-220-61-136.us-east-2.compute.amazonaws.com/racecar-ws.zip
cd racecar-ws
rm -rf build devel
catkin_make
```

racecar kodunu test et
```bash
source devel/setup.bash # veya zsh kullanıyorsan setup.zsh
roslaunch racecar teleop.launch
```

"Portları bulamadım" gibi bir yığın hata göreceksin. Bu hataları düzeltmemiz için port kuralları ayarlamamız gerek


### Usb port kuralları konfigürasyonu
Usb cihazları bul:
```bash
lsusb
```
Çıktı böyle birşey olacak:
```bash
Bus 002 Device 003: ID 0bda:0411 Realtek Semiconductor Corp.
Bus 002 Device 002: ID 0bda:0411 Realtek Semiconductor Corp.
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 010: ID 0403:6001 Future Technology Devices International, Ltd FT232 USB-Serial (UART) IC
Bus 001 Device 004: ID 045e:0745 Microsoft Corp. Nano Transceiver v1.0 for Bluetooth
Bus 001 Device 009: ID 046d:c21f Logitech, Inc. F710 Wireless Gamepad [XInput Mode]
Bus 001 Device 007: ID 046d:082d Logitech, Inc. HD Pro Webcam C920
Bus 001 Device 006: ID 1b4f:9d0f
Bus 001 Device 005: ID 0483:5740 STMicroelectronics STM32F407
Bus 001 Device 003: ID 0bda:5411 Realtek Semiconductor Corp.
Bus 001 Device 002: ID 0bda:5411 Realtek Semiconductor Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
Scanse Sweep Lidar'ı, vesc'yi ve imu'yu arıyoruz. Bu cihazları filtrele:
```bash
lsusb | grep "9d0f\|STMicro\|Future Technology"
```
Çıktı:
```bash
Bus 001 Device 010: ID 0403:6001 Future Technology Devices International, Ltd FT232 USB-Serial (UART) IC
Bus 001 Device 006: ID 1b4f:9d0f
Bus 001 Device 005: ID 0483:5740 STMicroelectronics STM32F407
```
Vendor ID ve/veya Product ID'ye ihtiyacımız var. 
We need the vendor id's and/or product id's of these devices. Neyse ki lsusb aşağıdaki format ile bu iki bilgiyi de veriyor:
```bash
Bus <bus number> Device <device number>: ID <vendor id>:<product id> <Device name>
```
Hadi onları alalım.
İlk cihaz `(Future Technology Devices International, Ltd FT232 USB-Serial (UART) IC)` scanse sweep lidar oluyor. İkincisi imu (9d0f göndermesini farkettin mi?). Son olarak(`STMicroelectronics`) is vesc. Mesela vesc'nin vendor ID'si `0483`, and product ID'si `5740`.

**Usb port kurallarını ayarla**
```bash
sudo nano /etc/udev/rules.d/99-usb-serial.rules
```
Böyle göründüğünden emin ol:
```bash
ATTRS{idVendor}=="0403", SYMLINK+="sweep"
ATTRS{idProduct}=="6015", SYMLINK+="sweep"

ATTRS{idVendor}=="1b4f", SYMLINK+="imu"
ATTRS{idProduct}=="9d0f", SYMLINK+="imu"

ATTRS{idVendor}=="0483", SYMLINK+="vesc"
ATTRS{idProduct}=="5740", SYMLINK+="vesc"
```
**Usb cihazları tekrar çıkar-tak**, ve bağlantıyı test et:
```bash
l /dev/vesc || l /dev/sweep || l /dev/imu
```
Bu kısayolların işaret ettiği diğer portları görmen gerek:
```bash
lrwxrwxrwx 1 root root 7 Nov  9 11:16 /dev/vesc -> ttyACM0
lrwxrwxrwx 1 root root 7 Nov  9 10:59 /dev/sweep -> ttyUSB0
lrwxrwxrwx 1 root root 7 Nov  8 21:29 /dev/imu -> ttyACM1
```
---------------------------------
Ek olarak bu da işe yarayabilir, fakat bu uygulama için ihtiyacımız yok:
```bash
# Cihazların seri numaralarını öğrenmek için:
usb-devices | grep "Manufacturer\|Product\|SerialNumber\|^$"

```


## ROS uzaktan bağlantısını varsayılan olarak açılması için konfigüre et
`nano ~/.profile` e git
Aşağıdaki satırları ekle:

 ---
 Uzak makinada (robot):
```bash
export ROS_MASTER_URI=http://$(echo -e $(hostname -I)):11311
export ROS_IP=$(hostname -I)
```
---
Yerel makinada (Bilgisayarın, veya muhtemelen sanal bilgisayarın):
```bash
export ROS_MASTER_URI=http://$(sudo arp-scan --localnet | grep NVIDIA | awk '{print $1;}'):11311
export ROS_IP=$(hostname -I)
```
---
`nano ~/.bashrc` ye git ve şu satırı **dosyanın sonuna değil, başına** ekle:
```bash
source ~/.profile
```
Eğer zsh kullanıyorsan, `nano ~/.zshrc` e git ve aşağıdaki satırları **dosyanın sonuna değil, başına** ekle:
```bash
emulate sh
. ~/.profile
emulate zsh
```
