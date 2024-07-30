# How To Make A Working TWRP Device Tree For Your MediaTek Device & Start Building Them, Online
>#### This Guide is tested on 64-bit mt6735/53 chipset device. It will also work on any 64-bit and 32-bit devices.

>You will need something from your Stock ROM first. Get them all and Try to modify it using the procedure.

## Table of contents
- [x] [Get Your Stock Recovery](#1-get-your-official-stock-recovery-and-stock-roms-buildprop)
- [x] [Extract The Recovery Into Ramdisk & Kernel](#2-extract-the-ramdisk-and-kernel-zimage)
- [x] [Create A GitHub Repository](#3-create-a-github-repository)
- [x] [Make Notes from build.prop](#4-make-note-of-these-things-from-buildprop)
- [x] [Add Some Stock Files](#5-put-some-stock-files-in-the-folder)
- [x] [Create _"Android.mk"_](#6-create-a-new-file-called-androidmk)
- [x] [Create _"AndroidProducts.mk"_](#7-create-a-new-file-called-androidproductsmk)
- [x] [Create _"BoardConfig.mk"_](#8-create-the-base-file-called-boardconfigmk)
- [x] [Create _"omni\_$$DEVICE$$.mk"_](#9-create-a-new-file-called-omni_devicemk-as-in-omni_primo_rx5mk)
- [x] [Publish the GitHub Repository](#10-publish-the-github-repository)
- [x] [Create An Account in Travis CI](#11-create-an-account-in-travis-ci-for-building-the-recovery)
- [x] [Create `.travis.yml` in GitHub Repo](#12-create-a-file-named-travisyml-in-github-repo)
- [x] [Fire-Up The Build](#13-fire-up-the-build)
- [x] [Flash The TWRP](#14-flash-and-enjoy)
- [x] [Troubleshooting](#15-troubleshooting-recovery-boot)

### 1. Get Your Official Stock Recovery and Stock ROM's __build.prop__
- You have to get your device's Stock `"recovery.img"` from the Official Firmware which you can _(or maybe can't)_ get from the device manufacturer's website.
- You have to get the `build.prop` file (from ROM's /system/build.prop) because there are a few key things that we need. 
  - Use ADB to Pull the file by the command `adb pull /system/build.prop`

### 2. Extract the __ramdisk__ and kernel __zImage__
- Use [`Carliv Image Kitchen`](https://gitlab.com/carliv/carliv_image_kitchen.git) to extract the image from Windows or Linux. 
- Need a Guide to Use it? [Read This](https://forum.xda-developers.com/android/development/tool-cika-carliv-image-kitchen-android-t3013658).

### 3. Create A GitHub Repository
- Go to https://github.com/ and SignUp there.
- You have to [Create an online GitHub Repository](https://github.com/new) with the name of the device.
  - For Example: something like `android_device_Walton_PrimoRX5` or `twrp_device_Walton_PrimoRX5` or whatever you want to call it.
- Go to https://github.com/settings/tokens/new/
  - It will take you to a page to create a New personal access token.
  - Name your Token as anything you like, for example: "GitHubToken" and save the Token Value in Notepad. We will need it later.

### 4. Make Note Of These Things From __build.prop__
- Write down the keys and their values in Notepad or something of the followings:
- For ease, I am showing from my device. 
  As in "Keys" = "Values"
  ```
  ro.build.product=gionee6735_65u_m0                        # Company Given Codename for the Device.
  ro.product.board=Primo_RX5                                # The Board Name for Build.
  ro.product.brand=WALTON                                   # The Brand and The Manufacturer
  ro.product.manufacturer=WALTON                              # can be of the Same Name.
  ro.product.name=Primo_RX5                                 # The Device Name and/or The Codename
  ro.product.device=Primo_RX5                                 # we will be using.
  ro.product.model=Primo RX5                                # A Nickname, we will ignore it.
  ro.mediatek.platform=MT6735                               # The Chipset Platform of the Project.
  ro.mediatek.project.path=device/gionee/gionee6735_65u_m0  # The Original Build Location from Company.
  ro.product.cpu.abi=arm64-v8a                              # Shows if the device is 6-bit or 32-bit.
  ```

### 5. Put Some Stock Files In The Folder
- Create a Folder named `prebuilt` and put the unpacked `recovery.img-kernel` inside it. Rename that file to `zImage`
- Create another Folder named `recovery`. Go in there and create a folder named `root`
- In the `root` folder, Put your `factory_init.*.rc` files, `meta_init.*.rc` files, `ueventd.mt****.rc`, `init.recovery.mt****.rc` files
- Make a new folder named `etc` inside `root`. Make a file named `recovery.fstab`
  - Use [THIS FILE](https://github.com/rokibhasansagar/android_device_twrp_WALTON_Primo_RX5/raw/master/recovery/root/etc/recovery.fstab) as a reference point and edit it.
  - You Have to edit the Mount Addresses (Like /dev/block/platform/mtk-msdc.0/11230000.msdc0/by-name/system), Alternate Short Mount Addresses (Like /dev/block/mmcblk0p20).
    >You can get Some of the Mount Addresses by ADB Commands from PC, using `adb shell cat /proc/self/mountstats` or `adb shell cat /proc/self/mounts` or `adb shell cat /proc/self/mountinfo`.
    >Also check whether the Basic FSType (emmc or ext4) of the partitions match with your devices. 
    >If you fail to know the Alternate Short Mount Addresses, Just erase them.
- Make a new folder named `sbin` inside `root`. Make a file named `permissive.sh` with File Permission set to "644" by using `chmod a+x permissive.sh` shell command
  - The file must contain the below content:
    ```sh
    #!/sbin/sh

    setenforce 0

    # Get your device's block path where "system", "recovery", etc. lives.
    # That can be "/dev/block/bootdevice/by-name" or something like that.
    mkdir -p /dev/block/platform/mtk-msdc.0/by-name/
    busybox mount -o bind /dev/block/platform/mtk-msdc.0/11230000.msdc0/by-name/ /dev/block/platform/mtk-msdc.0/by-name/
    ```
  The above script is for adding Old-Style Mount Points as-well-as New-Style Mount Points to avoid ROM Conflicts.

### 6. Create A New File Called _"Android.mk"_
- Make a new File, name it `Android.mk`, with the below content
  ```makefile
  # Replace $$DEVICE$$ with your Device Name's Value. Mine is Primo_RX5.
  # Replace $$BRAND$$ with your Brand's / Manufacturer's Value, Mine is WALTON 
  
  ifneq ($(filter $$DEVICE$$,$(TARGET_DEVICE)),)

  LOCAL_PATH := device/$$BRAND$$/$$DEVICE$$

  include $(call all-makefiles-under,$(LOCAL_PATH))

  endif
  ```

### 7. Create A New File Called _"AndroidProducts.mk"_
- Make a new File, name it `AndroidProducts.mk`, with the below content
  ```makefile
  # Replace $$DEVICE$$ with your Device Name's Value.
  # Replace $$BRAND$$ with your Brand's / Manufacturer's Value.
  # The part of last line in mine looks like "omni_Primo_RX5.mk"
  
  LOCAL_PATH := device/$$BRAND$$/$$DEVICE$$

  PRODUCT_MAKEFILES := $(LOCAL_PATH)/omni_$$DEVICE$$.mk
  ```

### 8. Create The Base File Called _"BoardConfig.mk"_
- Make a new File, name it `BoardConfig.mk`, with the below content
  ```makefile
  LOCAL_PATH := device/$$BRAND$$/$$DEVICE$$

  TARGET_BOARD_PLATFORM := mt6735               # From ro.mediatek.platform, but lowercase value
  TARGET_NO_BOOTLOADER := true
  TARGET_BOOTLOADER_BOARD_NAME := Primo_RX5     # From ro.product.board

  # These two are for MTK Chipsets only
  BOARD_USES_MTK_HARDWARE := true
  BOARD_HAS_MTK_HARDWARE := true

  # Recovery
  TARGET_USERIMAGES_USE_EXT4 := true
  TARGET_USERIMAGES_USE_F2FS := true            # To add info about F2FS Filesystem Data Block
  # Put The Size of your Recovery Partition below, get it from your "MT****_Android_scatter.txt"
  BOARD_RECOVERYIMAGE_PARTITION_SIZE := 16777216
  # BOARD_USES_FULL_RECOVERY_IMAGE := true      # Uncomment this line if you want to remove size restriction
  BOARD_FLASH_BLOCK_SIZE := 0                   # Might be different for your chip
  BOARD_HAS_NO_REAL_SDCARD := true              # Depricated
  # BOARD_HAS_NO_SELECT_BUTTON := true          # Depricated
  BOARD_SUPPRESS_SECURE_ERASE := true
  BOARD_HAS_NO_MISC_PARTITION := true           # Delete if your partition table has /misc
  BOARD_RECOVERY_SWIPE := true
  BOARD_USES_MMCUTILS := true
  BOARD_SUPPRESS_EMMC_WIPE := true
  BOARD_CHARGER_SHOW_PERCENTAGE := true
  RECOVERY_SDCARD_ON_DATA := true               # Optional: If /sdcard partition is emulated on /data partition 

  # TWRP stuff
  TW_EXCLUDE_SUPERSU := true                    # true/false: Add SuperSU or not
  TW_INCLUDE_CRYPTO := true                     # true/false: Add Data Encryption Support or not
  TW_INPUT_BLACKLIST := "hbtp_vm"               # Optional: Disables virtual mouse
  TW_SCREEN_BLANK_ON_BOOT := true
  TW_THEME := portrait_hdpi                     # Set the exact theme you wanna use. If resulation doesn't match, define the height/width
  DEVICE_RESOLUTION := 720x1280                 # The Resolution of your Device
  TARGET_SCREEN_HEIGHT := 1280                    # The height
  TARGET_SCREEN_WIDTH := 720                      # The width
  TARGET_RECOVERY_PIXEL_FORMAT := "RGBA_8888"
  # Set the Brightness Control File Path below (as per your chip/device)
  TW_BRIGHTNESS_PATH := /sys/class/leds/lcd-backlight/brightness
  TW_SECONDARY_BRIGHTNESS_PATH := /sys/devices/platform/leds-mt65xx/leds/lcd-backlight/brightness
  # Set the Path of Logical Units (LUNs) for Storage below (as per your chip/device)
  TARGET_USE_CUSTOM_LUN_FILE_PATH := /sys/devices/platform/mt_usb/musb-hdrc.0.auto/gadget/lun%d/file
  TARGET_USE_CUSTOM_LUN_FILE_PATH := /sys/class/android_usb/android0/f_mass_storage/lun/file
  TW_MAX_BRIGHTNESS := 255
  TW_DEFAULT_BRIGHTNESS := 80                   # Set custom brightness, low is better

  TW_INCLUDE_NTFS_3G := true                    # Include NTFS Filesystem Support
  TW_INCLUDE_FUSE_EXFAT := true                 # Include Fuse-ExFAT Filesystem Support
  TWRP_INCLUDE_LOGCAT := true                   # Include LogCat Binary
  TW_INCLUDE_FB2PNG := true                     # Include Screenshot Support
  TW_DEFAULT_LANGUAGE := en                     # Set Default Language 
  TW_EXTRA_LANGUAGES := false

  # Kernel
  TARGET_IS_64_BIT := true                      # true/false: Determine if the device is 64-bit or not
  TARGET_PREBUILT_KERNEL := $(LOCAL_PATH)/prebuilt/zImage
  TARGET_PREBUILT_RECOVERY_KERNEL := $(LOCAL_PATH)/prebuilt/zImage
  # Get the CMDLine, Base, Pagesize and offsets from Unpacked recovery image and put below
  BOARD_KERNEL_CMDLINE := bootopt=64S3,32N2,64N2 androidboot.selinux=permissive
  BOARD_KERNEL_BASE := 0x40078000
  BOARD_KERNEL_PAGESIZE := 2048
  BOARD_MKBOOTIMG_ARGS := --ramdisk_offset 0x03f88000 --tags_offset 0x0df88000
  
  # Set FSTAB
  TARGET_RECOVERY_FSTAB := $(LOCAL_PATH)/recovery/root/etc/recovery.fstab

  TARGET_BOARD_SUFFIX := _64                    # Remove if the device is 32-bit
  TARGET_USES_64_BIT_BINDER := true             # Remove if the device is 32-bit

  # Architecture
  # According to the device's architecture (64-bit or 32-bit)
  ifeq ($(TARGET_IS_64_BIT),true)
  TARGET_ARCH := arm64
  TARGET_ARCH_VARIANT := armv8-a
  TARGET_CPU_ABI := arm64-v8a
  TARGET_CPU_ABI2 :=
  TARGET_CPU_VARIANT := cortex-a53
  TARGET_2ND_ARCH := arm
  TARGET_2ND_ARCH_VARIANT := armv7-a-neon
  TARGET_2ND_CPU_ABI := armeabi-v7a
  TARGET_2ND_CPU_ABI2 := armeabi
  TARGET_2ND_CPU_VARIANT := cortex-a53
  TARGET_CPU_ABI_LIST_64_BIT := $(TARGET_CPU_ABI)
  TARGET_CPU_ABI_LIST_32_BIT := $(TARGET_2ND_CPU_ABI),$(TARGET_2ND_CPU_ABI2)
  TARGET_CPU_ABI_LIST := $(TARGET_CPU_ABI_LIST_64_BIT),$(TARGET_CPU_ABI_LIST_32_BIT)
  else
  TARGET_ARCH := arm
  TARGET_ARCH_VARIANT := armv7-a-neon
  TARGET_CPU_ABI := armeabi-v7a
  TARGET_CPU_ABI2 := armeabi
  TARGET_CPU_VARIANT := cortex-a7
  TARGET_CPU_ABI_LIST := $(TARGET_CPU_ABI),$(TARGET_CPU_ABI2)
  endif
  
  ```

### 9. Create A New File Called _"omni\_$$DEVICE$$.mk"_ as in "omni_Primo_RX5.mk"
- Make a new File, name it `omni_$$DEVICE$$.mk`, with the below content
  ```makefile
  $(call inherit-product, $(SRC_TARGET_DIR)/product/full_base.mk)
  
  # Add this line if your device is 64-bit
  $(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
  # Otherwise, If you have 32-bit device, add the below line instead of above line
  $(call inherit-product, $(SRC_TARGET_DIR)/product/core_minimal.mk
  
  # Another common config inclusion
  $(call inherit-product, $(SRC_TARGET_DIR)/product/embedded.mk)
  
  # If you are building from OmniROM's minimal source, Inherit some common Omni stuff.
  $(call inherit-product, vendor/omni/config/common.mk)

  # Replace $$DEVICE$$ with your Device Name's Value.
  # Replace $$BRAND$$ with your Brand's / Manufacturer's Value.
  PRODUCT_COPY_FILES += device/$$BRAND$$/$$DEVICE$$/prebuilt/zImage:kernel
  # Fles under $(LOCAL_PATH)/recovery/root/ gets automatically copied into recovery
  # PRODUCT_COPY_FILES += $(LOCAL_PATH)/recovery/root/*:root/*

  PRODUCT_DEVICE := $$DEVICE$$
  PRODUCT_NAME := omni_$$DEVICE$$
  PRODUCT_BRAND := $$BRAND$$
  PRODUCT_MODEL := $$DEVICE$$
  PRODUCT_MANUFACTURER := $$BRAND$$

  # Forcefully add mtp support (adb is already there)
  PRODUCT_DEFAULT_PROPERTY_OVERRIDES += \
      persist.sys.usb.config=mtp
  
  # Add fingerprint from Stock ROM build.prop
  PRODUCT_BUILD_PROP_OVERRIDES += \
      # These lines are from my device. You MUST Replace yours.
      BUILD_FINGERPRINT="WALTON/Primo_RX5/Primo_RX5:6.0/MRA58K/1465782828:user/release-keys" \
      PRIVATE_BUILD_DESC="full_gionee6735_65u_m0-user 6.0 MRA58K 1465782828 release-keys"
  ```

### 10. Publish the GitHub Repository
- If you have done the above process entirely online inside github.com, then it is okay.
- But, if you have done the above things Locally on your PC, Just upload the entire Folder or `git push` to server.
  >No detail or tutorial will be provided here. This is A-B-C of Git, Man!

### 11. Create An Account in Travis CI for Building the Recovery
- Go to https://travis-ci.org/ and SignUp there using GitHub.
- Go to https://travis-ci.org/profile/ and Add the Repository there by tapping on the Radio Button switch at the right side of the Repository name.
- Now, On its further right corner, you will find an individual repository `Settings` button. Click it. You will be on the Repository's Settings Page Now.
  - Go below where "Environment Variables" is written.
  - Add "GitOAUTHToken" as New Environment Variables' _Name_ and put your GitHub Token's value as it's _Value_ here.
  

### 12. Create A File Named `.travis.yml` in GitHub Repo
- To Start the Build Process, go to your created TWRP Device Tree in GitHub.
- Make a file named `.travis.yml` with below content
  ```yaml
  os: linux
  dist: bionic
  language: generic
  sudo: required
  git:
    depth: 1
  addons:
    apt:
      update:
        - true
  services:
    - docker
  before_install:
    - docker pull fr3akyphantom/droid-builder:latest
  before_script:
    - cd $HOME && mkdir twrp
    # download the TWRP Compressed Source Files from PhantomZone54's Release
    # Uncomment & Use below line If Building for Lollipop-based Devices
    # - TWRP_SOURCE="https://github.com/PhantomZone54/twrp_sources_norepo/releases/download/v3.3.1-20200222/MinimalOmniRecovery-twrp-5.1-norepo-20200222.tar.xz"
    # Use below line If Building for Marshmallow-based Devices
    - TWRP_SOURCE="https://github.com/PhantomZone54/twrp_sources_norepo/releases/download/v3.3.1-20200222/MinimalOmniRecovery-twrp-6.0-norepo-20200222.tar.xz"
    # Uncomment & Use below line If Building for Nougat-based Devices
    # - TWRP_SOURCE="https://github.com/PhantomZone54/twrp_sources_norepo/releases/download/v3.3.1-20200222/MinimalOmniRecovery-twrp-7.1-norepo-20200222.tar.xz"
    - wget -q ${TWRP_SOURCE} -O $HOME/twrp.tar.xz
    - tar -xJf twrp.tar.xz --directory $HOME/twrp/ && rm twrp.tar.xz
  script:
    # Replace your $$USERNAME$$, $$REPO_URL$$, $$BRAND$$, $$DEVICE$$
    - cd $HOME/twrp/ && git clone https://github.com/$$USERNAME$$/$$REPO_URL$$.git device/$$BRAND$$/$$DEVICE$$
    - rm -rf bootable/recovery && git clone https://github.com/omnirom/android_bootable_recovery -b android-9.0 --depth 1 bootable/recovery
    - |
      docker run --rm -i -e USER_ID=$(id -u) -e GROUP_ID=$(id -g) -v "$(pwd):/home/builder/twrp/:rw,z" -v "${HOME}/.ccache:/srv/ccache:rw,z" fr3akyphantom/droid-builder bash << EOF
      cd /home/builder/twrp/
      source build/envsetup.sh
      # Choose build flavor as "eng" or "userdebug"
      BUILD_FLAVOR="eng"
      lunch omni_$$DEVICE$$-${BUILD_FLAVOR}
      make -j$(nproc --all) recoveryimage
      exit
      EOF
  after_success:
    - export version=$(cat bootable/recovery/variables.h | grep "define TW_MAIN_VERSION_STR" | cut -d '"' -f2)
    - cp $HOME/twrp/out/target/product/$$DEVICE$$/recovery.img $HOME/twrp/TWRP-$version-$$DEVICE$$-$(date +"%Y%m%d")-Unofficial.img
    - cd $HOME/twrp/
    # Optional: You might need to switch from https://transfer.sh to https://file.io
    # - curl -s --upload-file TWRP-$version-$$DEVICE$$-$(date +"%Y%m%d")-Unofficial.img https://transfer.sh/ && echo ""
  deploy:
    skip_cleanup: true
    provider: releases
    # The secret api_key will be loaded from the environment variables
    api_key: $GitOAUTHToken
    file_glob: true
    file: $HOME/twrp/*.img
    on:
      tags: false
      repo: $$USERNAME$$/$$REPO_URL$$ # Optional: If you want to deploy on different repository
      branch: master # Optional: Needs to be exact as the config branch
  branches:
    only:
      - master # Set travis builder branch(es) names
    except:
      - /^(?i:untagged)-.*$/
      - /^v\d+\.\d+(\.\d+)?(-\S*)?$/
  ```

### 13. Fire-Up The Build
- When you save the .travis.yml file for the first time, the build will automatically start.
  - You can see the progress of the build on https://travis-ci.org/$$USERNAME$$/$$REPO_URL$$/
- After the first build, you can see the result on the GitHub Repository's Releases Page at https://github.com/$$USERNAME$$/$$REPO_URL$$/releases/
  - You can also download the file from viewing the Travis CI log, where the script uploaded it into https://transfer.sh/

### 14. Flash and Enjoy
- Flash the newly-build Recovery. It is better Not to Use with Stock ROM.

### 15. Troubleshooting Recovery Boot
- If The Recovery does not Boot,
  - [x] Unpack the Stock Recovery Image again, and double-check properties used in TWRP Tree
  - [x] Double-check files that are included in the Tree
- If the Recovery starts Upside-Down,
  - [x] Edit BoardConfig.mk and insert the below line before the `include` part (last line)
    ```makefile
    BOARD_HAS_FLIPPED_SCREEN := true
    ```
