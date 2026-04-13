# -*- mode: ruby -*-
# vi: set ft=ruby :

# ================================================
# 系統計時器：計算 vagrant up 總耗時
# ================================================
start_time = Time.now

at_exit do
  end_time = Time.now
  duration = end_time - start_time
  minutes = (duration / 60).to_i
  seconds = (duration % 60).round(2)
  puts "\n========================================================"
  puts ">>> [系統提示] 本次 Vagrant 執行完畢！"
  puts ">>> 總耗時：#{minutes} 分鐘 #{seconds} 秒"
  puts "========================================================\n"
end

# ================================================
# Testbed前置作業：自動處理 SSH 金鑰，Ansible會用到
# ================================================
key_path = File.join(__dir__, "lab_key")
pub_key_path = "#{key_path}.pub"

# 如果偵測到私鑰不存在，就呼叫系統指令自動產生
unless File.exist?(key_path)
  puts ">>> [系統提示] 偵測到首次執行，正在為您的實驗室自動產生專屬 SSH 金鑰..."
  # 呼叫 ssh-keygen 產生金鑰
  # 使用 RSA 以確保最大相容性，且不設定密碼 (-N "")
  system("ssh-keygen -t rsa -b 2048 -f \"#{key_path}\" -q -N \"\"")
end

# 讀取公鑰內容到變數中，準備發派給所有靶機，Ansbile的SSH要用
LAB_PUB_KEY = File.read(pub_key_path).strip


# ================================================
# 全局變數與環境設定區
# ================================================
#START_WINDOWS 是用來控制vagrant up時，要不要帶Windows起來的一個變數，
#設為false，vagrant up的時候，不會去帶Windows的機器
START_WINDOWS = true
# IP 前綴變數，方便統一修改網段
IP_PREFIX = "10.0.0."
# 💡 強制關閉平行啟動，確保 16 台機器嚴格依照由上到下的順序乖乖排隊開機
ENV["VAGRANT_NO_PARALLEL"] = "yes"


# ================================================
# Vagrantfile 正文
# ================================================

Vagrant.configure("2") do |config|

  # 在最上方統一設定：只要是 VirtualBox 啟動的機器，全部都用link_clone！
  # 會先在virtualbox做一份BASE的VM，其他機器再去link那台base vm
  config.vm.provider "virtualbox" do |v|
    v.linked_clone = true
  end
 
  # ==========================================
  # 建立 Linux 靶機清單，精準派發 SSH 公鑰 
  # ==========================================
  linux_vms = ["kali", "centos", "rocky9", "ubuntu20", "ubuntu22", "ubuntu24", "ansible_control", "ubuntu12", "ubuntu14"]

  linux_vms.each do |vm_name|
    config.vm.define vm_name do |node|
      node.vm.provision "shell", inline: <<-SHELL
        echo ">>> [系統設定] 正在為 #{vm_name} 植入專屬 SSH 公鑰..."
        mkdir -p /home/vagrant/.ssh
        echo "#{LAB_PUB_KEY}" >> /home/vagrant/.ssh/authorized_keys
        chmod 700 /home/vagrant/.ssh
        chmod 600 /home/vagrant/.ssh/authorized_keys
        chown -R vagrant:vagrant /home/vagrant/.ssh
      SHELL
    end
  end
 
 
  # ==========================================
  # 節點 1: Kali Linux (攻擊機)
  # ==========================================
  config.vm.define "kali" do |kali|
    kali.vm.box = "kalilinux/rolling"
    kali.vm.hostname = "kali-2026-01-231"
    #不指定網卡，vagrant會用選單，問說要bridge到哪一張網卡
    #kali.vm.network "public_network"   
    #kali.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    kali.vm.network "public_network", ip: "#{IP_PREFIX}231", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    kali.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Kali"
      vb.memory = "2048"
      vb.cpus = 2
      vb.gui = false
      #設定這只有這台是Link Clone
      #vb.linked_clone = true 
    end

    # 在Kali新增kali這個User，並配置好所有權限與金鑰
    kali.vm.provision "shell", inline: <<-SHELL
      echo ">>> [系統設定] 檢查使用者 kali 是否存在..."
      
      if id "kali" >/dev/null 2>&1; then
        echo ">>> 使用者 kali 已經存在，跳過建立流程。"
      else
        echo ">>> 偵測到無此帳號，開始建立超級使用者 kali..."
        useradd -m -s /bin/bash -G sudo kali
        echo "kali:kali" | chpasswd
        
        # 💡 架構師加碼：設定免密碼 sudo
        # 確保使用者在打靶或裝軟體時，不需要一直瘋狂輸入密碼
        echo "kali ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/kali
      fi
      
      # ==========================================
      # 💡 確保 kali 帳號也擁有外部注入的 SSH 公鑰
      # ==========================================
      echo ">>> [系統設定] 正在同步 kali 使用者的 SSH 金鑰..."
      mkdir -p /home/kali/.ssh
      
      # 檢查公鑰是否已經存在，避免重複寫入 (確保冪等性)
      if ! grep -q "#{LAB_PUB_KEY}" /home/kali/.ssh/authorized_keys 2>/dev/null; then
        echo "#{LAB_PUB_KEY}" >> /home/kali/.ssh/authorized_keys
      fi
      
      # 嚴格校正權限，防止 SSH 伺服器因為權限太寬鬆而拒絕連線
      chmod 700 /home/kali/.ssh
      chmod 600 /home/kali/.ssh/authorized_keys
      chown -R kali:kali /home/kali/.ssh
      
      echo ">>> 使用者 kali 環境就緒！"
    SHELL
    
  end

  # ==========================================
  # 節點 2: Ansible Control Node (控制機)
  # ==========================================
  config.vm.define "ansible_control" do |node|
    # 沿用現有的 Ubuntu 22.04 映像檔
    node.vm.box = "bento/ubuntu-24.04"
    node.vm.hostname = "ansible-control-232"
    node.vm.network "public_network", ip: "#{IP_PREFIX}232", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ansible_Control"
      vb.memory = "1024" # 作為純控制節點，1GB 記憶體通常已非常充足
      vb.cpus = 1
      vb.gui = false
    end
    
    # 1. 直接複製「一對」金鑰檔案 (最推薦的標準作法)
    node.vm.provision "file", source: "lab_key", destination: "/home/vagrant/.ssh/id_rsa"
    node.vm.provision "file", source: "lab_key.pub", destination: "/home/vagrant/.ssh/id_rsa.pub"
    
    # 透過 Shell Provisioner 在開機時全自動安裝最新版 Ansible
    node.vm.provision "shell", inline: <<-SHELL
      echo ">>> [系統設定] 正在同步金鑰權限與安裝 Ansible..."     
      # 私鑰 600 (只有我能讀寫)
      chmod 600 /home/vagrant/.ssh/id_rsa
      # 公鑰 644 (大家都能讀，但我能寫)
      chmod 644 /home/vagrant/.ssh/id_rsa.pub
      chown -R vagrant:vagrant /home/vagrant/.ssh
      # 檢查 ansible 指令是否已存在 (這是實現冪等性的核心判斷)
      if command -v ansible >/dev/null 2>&1; then
        echo ">>> Ansible 已經安裝過了，跳過安裝流程。"
        ansible --version | head -n 1
      else
        echo ">>> 偵測到尚未安裝 Ansible，開始執行「黃金版本」安裝..."
        export DEBIAN_FRONTEND=noninteractive
        
        apt-get update
        
        # 1. 捨棄 PPA，改裝 Python 官方套件管理員 pip3
        apt-get install -y python3-pip
        apt-get install -y python3-passlib
        
        # 2. 透過 pip3 強制鎖定安裝 Ansible 9.13.0 (Core 2.16.14)
        # 💡 加入 --break-system-packages 以突破 Ubuntu 24.04 的系統保護限制
        pip3 install --ignore-installed "ansible-core==2.16.14" "ansible==9.13.0" --break-system-packages
        pip3 install --upgrade --ignore-installed "ntlm-auth" "pywinrm" --break-system-packages
        pip3 install --upgrade --ignore-installed "paramiko" --break-system-packages
        echo ">>> Ansible 黃金版本 (Core 2.16.14) 安裝完畢！"
      fi

      # 判斷 .ansible.cfg 是否已經存在
      if [ ! -f /home/vagrant/.ansible.cfg ]; then
        echo ">>> 找不到設定檔，建立全新的 .ansible.cfg 並關閉 Strict Host Key Checking..."
        echo -e "[defaults]\nhost_key_checking = False" > /home/vagrant/.ansible.cfg
        chown vagrant:vagrant /home/vagrant/.ansible.cfg
      else
        echo ">>> .ansible.cfg 已存在，跳過覆蓋動作，保留您的自訂設定。"
      fi
    
      # =========================================================
      # 💡 關鍵新增：自動喚醒 OpenSSL 的 Legacy 模組 (支援 MD4)
      # 針對被完全刪除設定的 Ubuntu 24.04 進行強行注入
      # =========================================================
      echo ">>> [系統設定] 正在修改 OpenSSL 設定以支援 Windows NTLM 驗證..."
      cp /etc/ssl/openssl.cnf /etc/ssl/openssl.cnf.bak
      
      # 檢查是否已經注入過 (確保冪等性，重複執行也不會壞)
      if ! grep -q "legacy = legacy_sect" /etc/ssl/openssl.cnf; then
          # 1. 在 [provider_sect] 下方強行插入 legacy 宣告
          sed -i '/^\[provider_sect\]/a legacy = legacy_sect' /etc/ssl/openssl.cnf
          
          # 2. 將被註解的 activate = 1 解開 (為了啟動 default_sect)
          sed -i 's/^# activate = 1/activate = 1/' /etc/ssl/openssl.cnf
          
          # 3. 在檔案最尾端，追加 [legacy_sect] 的完整設定
          echo -e "\n[legacy_sect]\nactivate = 1" >> /etc/ssl/openssl.cnf
          echo ">>> [系統設定] OpenSSL Legacy 模組強行注入成功！"
      else
          echo ">>> [系統設定] OpenSSL Legacy 模組已存在，跳過注入。"
      fi
      # =========================================================
      
    
      # 💡 新增這段 Shell Provisioner 來安裝工具與 Clone 專案
      echo ">>> [系統設定] 正在更新套件庫並安裝 net-tools 與 git..."
    
      export DEBIAN_FRONTEND=noninteractive
      
      # 1. 更新 apt 索引並安裝軟體 (-y 代表遇到詢問自動回答 yes)
      apt-get update
      apt-get install -y net-tools git

      # =========================================================
      # 💡 關鍵新增：自動喚醒 OpenSSL 的 Legacy 模組 (支援 MD4)
      # =========================================================
      #echo ">>> [系統設定] 正在修改 OpenSSL 設定以支援 Windows NTLM 驗證..."
      # 備份原檔，養成好習慣
      #cp /etc/ssl/openssl.cnf /etc/ssl/openssl.cnf.bak
      # 1. 解開 provider 區段的 legacy 宣告
      #sed -i 's/^# legacy = legacy_sect/legacy = legacy_sect/' /etc/ssl/openssl.cnf
      # 2. 解開 [legacy_sect] 標籤
      #sed -i 's/^# \[legacy_sect\]/\[legacy_sect\]/' /etc/ssl/openssl.cnf
      # 3. 精準解開 [legacy_sect] 下一行的 activate = 1
      #sed -i '/^\[legacy_sect\]/{n;s/^# activate = 1/activate = 1/}' /etc/ssl/openssl.cnf
      # =========================================================

      echo ">>> [系統設定] 正在從 GitHub Clone 專案..."
      
      # 2. 切換到 vagrant 使用者的家目錄
      cd /home/vagrant
      
      # 3. 執行 git clone
      git clone https://github.com/HelloNiHowMa/Config_Testbed
      
      # 4. 關鍵步驟：把下載下來的資料夾擁有者，從 root 改回 vagrant
      chown -R vagrant:vagrant /home/vagrant/Config_Testbed
      
      echo ">>> [系統設定] 初始化完成！"
    SHELL

  end
  

  # ==========================================
  # 節點 3: CentOS Stream 9 (靶機 1)
  # ==========================================
  config.vm.define "centos" do |centos|
    centos.vm.box = "bento/centos-stream-9"
    centos.vm.hostname = "centos9-233"
    centos.vm.network "public_network", ip: "#{IP_PREFIX}233", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"

    centos.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_CentOS9"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
    
  end

  # ==========================================
  # 節點 4: Rocky Linux 9 (靶機 2)
  # ==========================================
  config.vm.define "rocky9" do |node|
    node.vm.box = "bento/rockylinux-9"
    node.vm.hostname = "rocky9-234"
    node.vm.network "public_network", ip: "#{IP_PREFIX}234", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Rocky9"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 5: Ubuntu 20.04 (靶機 3)
  # ==========================================
  config.vm.define "ubuntu20" do |node|
    node.vm.box = "bento/ubuntu-20.04"
    node.vm.hostname = "ubuntu20-235"
    node.vm.network "public_network", ip: "#{IP_PREFIX}235", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu20"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 6: Ubuntu 22.04 (靶機 4)
  # ==========================================
  config.vm.define "ubuntu22" do |node|
    node.vm.box = "bento/ubuntu-22.04"
    node.vm.hostname = "ubuntu22-236"
    node.vm.network "public_network", ip: "#{IP_PREFIX}236", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu22"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 7: Ubuntu 24.04 (靶機 5)
  # ==========================================
  config.vm.define "ubuntu24" do |node|
    node.vm.box = "bento/ubuntu-24.04"
    node.vm.hostname = "ubuntu24-237"
    node.vm.network "public_network", ip: "#{IP_PREFIX}237", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu24"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 8: Windows 10 (靶機 6 - 用戶端)
  # ==========================================
  #config.vm.define "win10" do |node|
  config.vm.define "win10", autostart: START_WINDOWS do |node|
    # 使用社群中最穩定的 Windows 映像檔提供者
    node.vm.box = "gusztavvargadr/windows-10"
    node.vm.network "public_network", ip: "#{IP_PREFIX}238", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win10"
      vb.memory = "4096" # Windows 至少需要 4GB 記憶體才會順
      vb.cpus = 2
      vb.gui = false
    end
    
    
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to win10-alone-238..."
      Rename-Computer -NewName win10-alone-238 -Force -ErrorAction SilentlyContinue

    SHELL

    node.vm.provision :reload
 
  end

  # ==========================================
  # 節點 9: Windows 11 (靶機 7 - 用戶端)
  # ==========================================
  #config.vm.define "win11" do |node|
  config.vm.define "win11", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-11"
    node.vm.network "public_network", ip: "#{IP_PREFIX}239", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win11"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
    
    
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to win11-alone-239..."
      Rename-Computer -NewName win11-alone-239 -Force
    SHELL

    node.vm.provision :reload
  
  end

  # ==========================================
  # 節點 10: Windows Server 2022 (DC1 - 網域控制站預備機)
  # ==========================================
  config.vm.define "win2022_dc", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-server-2022-standard"
    #node.vm.hostname = "DC1-201"
    # 💡 直接在 public_network 指定固定 IP，並橋接到 Wi-Fi 網卡
    node.vm.network "public_network", ip: "#{IP_PREFIX}201", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win2022_DC1"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
            
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to DC1-201..."
      Rename-Computer -NewName DC1-201 -Force
    SHELL

    # 💡 關鍵順序：先重開機！讓 Windows 重新判定網路完畢
    node.vm.provision :reload

  end

  # ==========================================
  # 節點 11: Windows Server 2022 (SRV1 - 網域成員伺服器 1)
  # ==========================================
  config.vm.define "win2022_srv1", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-server-2022-standard"
    #node.vm.hostname = "SRV1-202"
    node.vm.network "public_network", ip: "#{IP_PREFIX}202", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win2022_SRV1"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
    
        
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to SRV1-202..."
      Rename-Computer -NewName SRV1-202 -Force
    SHELL

    # 💡 關鍵順序：先重開機！讓 Windows 重新判定網路完畢
    node.vm.provision :reload
    
  end

  # ==========================================
  # 節點 12: Windows Server 2022 (SRV2 - 網域成員伺服器 2)
  # ==========================================
  config.vm.define "win2022_srv2", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-server-2022-standard"
    #node.vm.hostname = "SRV2-203"
    node.vm.network "public_network", ip: "#{IP_PREFIX}203", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win2022_SRV2"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
    
    
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to SRV2-203..."
      Rename-Computer -NewName SRV2-203 -Force
    SHELL

    # 💡 關鍵順序：先重開機！讓 Windows 重新判定網路完畢
    node.vm.provision :reload
    
  end

  # ==========================================
  # 節點 13: Windows 10 (用戶端 2 - 固定 IP 版)
  # ==========================================
  config.vm.define "win10_dc_client", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-10"
    #node.vm.hostname = "win10-dc-client-210"
    # 指定固定 IP 為 10.0.0.210
    node.vm.network "public_network", ip: "#{IP_PREFIX}210", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win10_dc_client"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
    
    
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to win10-dc-210..."
      Rename-Computer -NewName win10-dc-210 -Force
    SHELL

    # 💡 關鍵順序：先重開機！讓 Windows 重新判定網路完畢
    node.vm.provision :reload

  end

  # ==========================================
  # 節點 14: Windows 11 (用戶端 2 - 固定 IP 版)
  # ==========================================
  config.vm.define "win11_dc_client", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-11"
    #node.vm.hostname = "win11-dc-client-211"
    # 指定固定 IP 為 10.0.0.211
    node.vm.network "public_network", ip: "#{IP_PREFIX}211", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win11_dc_client"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end
    
    
    node.vm.provision "shell", inline: <<-'SHELL'
      Write-Host ">>> [System] Disabling Network Discovery popup..."
      $regPath = "HKLM:\System\CurrentControlSet\Control\Network\NewNetworkWindowOff"
      if (!(Test-Path $regPath)) {
          New-Item -Path $regPath -Force | Out-Null
      }
      Write-Host ">>> [System] Renaming computer to win11-dc-211..."
      Rename-Computer -NewName win11-dc-211 -Force
    SHELL

    # 💡 關鍵順序：先重開機！讓 Windows 重新判定網路完畢
    node.vm.provision :reload

    
  end
 
        
  # ==========================================
  # 節點 15: Ubuntu 12.04 (靶機 9 - 極度老舊漏洞環境，如 bWAPP 原生相容)
  # ==========================================
  config.vm.define "ubuntu12" do |node|
    node.vm.box = "bento/ubuntu-12.04"
    node.vm.hostname = "ubuntu12-target-240"
    node.vm.network "public_network", ip: "#{IP_PREFIX}240", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu12"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 16: Ubuntu 14.04 (靶機 10 - 舊時代 LAMP 架構測試用)
  # ==========================================
  config.vm.define "ubuntu14" do |node|
    node.vm.box = "bento/ubuntu-14.04"
    node.vm.hostname = "ubuntu14-target-241"
    node.vm.network "public_network", ip: "#{IP_PREFIX}241", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu14"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end
  # ==========================================
  # 節點 17: Metasploitable3 - Ubuntu 14.04 (經典漏洞大補帖 Linux)
  # ==========================================
  config.vm.define "ms3_ub1404" do |node|
    node.vm.box = "rapid7/metasploitable3-ub1404"
    node.vm.hostname = "metasploitable3-ub1404"
    
    # 預設帳密設定 (MS3 的傳統)
    config.ssh.username = 'vagrant'
    config.ssh.password = 'vagrant'

    # 融入實驗室網段，指派 IP 結尾為 242
    node.vm.network "public_network", ip: "#{IP_PREFIX}242", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_MS3_Ubuntu1404"
      vb.memory = "2048"
      vb.cpus = 2
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 18: Metasploitable3 - Windows Server 2008 (經典漏洞大補帖 Windows)
  # ==========================================
  # 💡 套用 START_WINDOWS 變數控制是否啟動
  config.vm.define "ms3_win2k8", autostart: START_WINDOWS do |node|
    node.vm.box = "rapid7/metasploitable3-win2k8"
    node.vm.hostname = "metasploitable3-win2k8"
    
    # WinRM 通訊設定
    node.vm.communicator = "winrm"
    node.winrm.retry_limit = 60
    node.winrm.retry_delay = 10

    # 融入實驗室網段，指派 IP 結尾為 243
    node.vm.network "public_network", ip: "#{IP_PREFIX}243", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"

    # 💡 轉換為 VirtualBox 語法 (原先為 libvirt)
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_MS3_Win2008"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = false
    end

    # 配置防火牆以開放漏洞服務 (保留原 MS3 官方邏輯)
    case ENV['MS3_DIFFICULTY']
      when 'easy'
        node.vm.provision :shell, inline: "C:\\startup\\disable_firewall.bat"
      else
        node.vm.provision :shell, inline: "C:\\startup\\enable_firewall.bat"
        node.vm.provision :shell, inline: "C:\\startup\\configure_firewall.bat"
    end

    # 執行 MS3 內部預寫好的啟動腳本與清理作業
    node.vm.provision :shell, inline: "C:\\startup\\install_share_autorun.bat"
    node.vm.provision :shell, inline: "C:\\startup\\setup_linux_share.bat"
    #node.vm.provision :shell, inline: "rm C:\\startup\\*" 
  end
end
