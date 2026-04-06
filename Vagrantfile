# ==========================================
# 前置作業：自動處理 SSH 金鑰
# ==========================================
key_path = File.join(__dir__, "lab_key")
pub_key_path = "#{key_path}.pub"

# 如果偵測到私鑰不存在，就呼叫系統指令自動產生
unless File.exist?(key_path)
  puts ">>> [系統提示] 偵測到首次執行，正在為您的實驗室自動產生專屬 SSH 金鑰..."
  # 呼叫學生電腦 (Windows/Mac) 內建的 ssh-keygen 產生金鑰
  # 使用 RSA 以確保最大相容性，且不設定密碼 (-N "")
  system("ssh-keygen -t rsa -b 2048 -f \"#{key_path}\" -q -N \"\"")
end

# 讀取公鑰內容到變數中，準備發派給所有靶機
LAB_PUB_KEY = File.read(pub_key_path).strip

#START_WINDOWS 是一個變數，等下設定Windows VM的時候會參考它
START_WINDOWS = false

Vagrant.configure("2") do |config|

  # 在最上方統一設定：只要是 VirtualBox 啟動的機器，全部都用連結複製！
  # 會先用BOX做一份BASE的VM，再去link那台base vm
  #config.vm.provider "virtualbox" do |v|
  #  v.linked_clone = true
  #end
  
  # ==========================================
  # 節點 1: Kali Linux (攻擊機)
  # ==========================================
  config.vm.define "kali" do |kali|
    kali.vm.box = "kalilinux/rolling"
    kali.vm.hostname = "kali-attacker"
	#不指定網卡，vagrant會用選單，問說要bridge到哪一張網卡
	#kali.vm.network "public_network"   
    kali.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
	
	# 派發公鑰給靶機
    kali.vm.provision "shell", inline: <<-SHELL
      echo ">>> [系統設定] 正在植入實驗室專屬 SSH 公鑰..."
      
      # 確保 .ssh 資料夾存在
      mkdir -p /home/vagrant/.ssh
      
      # 將 Ruby 變數 LAB_PUB_KEY 的內容寫入 authorized_keys
      echo "#{LAB_PUB_KEY}" >> /home/vagrant/.ssh/authorized_keys
      
      # 確保權限正確 (SSH 對權限要求非常嚴格，錯了就連不上)
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
	  
	  echo ">>> [系統設定] 檢查使用者 kali 是否存在..."
      
      # 判斷系統中是否已經有 kali 這個帳號
      if id "kali" >/dev/null 2>&1; then
        echo ">>> 使用者 kali 已經存在，跳過建立流程。"
      else
        echo ">>> 偵測到無此帳號，開始建立超級使用者 kali..."
        
        # 1. 建立使用者 (-m 建立家目錄, -s 指定 bash, -G 加入 sudo 群組)
        useradd -m -s /bin/bash -G sudo kali
        
        # 2. 設定密碼為 kali (透過 chpasswd 達成非互動式修改)
        echo "kali:kali" | chpasswd
        
        echo ">>> 使用者 kali 建立與權限設定完畢！"
      fi
    SHELL
	
    kali.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Kali"
      vb.memory = "2048"
      vb.cpus = 2
      vb.gui = true
	  #設定這只有這台是Link Clone
	  #vb.linked_clone = true 
    end
  end

  # ==========================================
  # 節點 2: CentOS Stream 9 (靶機 1)
  # ==========================================
  config.vm.define "centos" do |centos|
    centos.vm.box = "bento/centos-stream-9"
    centos.vm.hostname = "centos-target"
	#centos.vm.network "public_network"
    centos.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"

    centos.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_CentOS9"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = true
    end
  end

  # ==========================================
  # 節點 3: Rocky Linux 9 (靶機 2)
  # ==========================================
  config.vm.define "rocky9" do |node|
    node.vm.box = "bento/rockylinux-9"
    node.vm.hostname = "rocky9-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Rocky9"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 4: Ubuntu 20.04 (靶機 3)
  # ==========================================
  config.vm.define "ubuntu20" do |node|
    node.vm.box = "bento/ubuntu-20.04"
    node.vm.hostname = "ubuntu20-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu20"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 5: Ubuntu 22.04 (靶機 4)
  # ==========================================
  config.vm.define "ubuntu22" do |node|
    node.vm.box = "bento/ubuntu-22.04"
    node.vm.hostname = "ubuntu22-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu22"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 6: Ubuntu 24.04 (靶機 5)
  # ==========================================
  config.vm.define "ubuntu24" do |node|
    node.vm.box = "bento/ubuntu-24.04"
    node.vm.hostname = "ubuntu24-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Ubuntu24"
      vb.memory = "1024"
      vb.cpus = 1
      vb.gui = false
    end
  end

  # ==========================================
  # 節點 7: Windows 10 (靶機 6 - 用戶端)
  # ==========================================
  #config.vm.define "win10" do |node|
  config.vm.define "win10", autostart: START_WINDOWS do |node|
    # 使用社群中最穩定的 Windows 映像檔提供者
    node.vm.box = "gusztavvargadr/windows-10"
    node.vm.hostname = "win10-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win10"
      vb.memory = "4096" # Windows 至少需要 4GB 記憶體才會順
      vb.cpus = 2
      vb.gui = true      # Windows 強烈建議開啟 GUI
    end
  end

  # ==========================================
  # 節點 8: Windows 11 (靶機 7 - 用戶端)
  # ==========================================
  #config.vm.define "win11" do |node|
  config.vm.define "win11", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-11"
    node.vm.hostname = "win11-target"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win11"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = true
    end
  end

  # ==========================================
  # 節點 9: Windows Server 2022 (靶機 8 - 網域控制站預備機)
  # ==========================================
  #config.vm.define "win2022" do |node|
  config.vm.define "win2022", autostart: START_WINDOWS do |node|
    node.vm.box = "gusztavvargadr/windows-server-2022-standard"
    node.vm.hostname = "win2022-dc"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
    node.vm.provider "virtualbox" do |vb|
      vb.name = "Testbed_Win2022"
      vb.memory = "4096"
      vb.cpus = 2
      vb.gui = true
    end
  end

  # ==========================================
  # 節點 10: Ansible Control Node (控制機)
  # ==========================================
  config.vm.define "ansible_control" do |node|
    # 沿用現有的 Ubuntu 22.04 映像檔
    node.vm.box = "bento/ubuntu-22.04"
    node.vm.hostname = "ansible-control"
    #node.vm.network "public_network"
    node.vm.network "public_network", bridge: "Intel(R) Wi-Fi 6E AX211 160MHz"
    
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
        echo ">>> 偵測到尚未安裝 Ansible，開始執行安裝..."
        export DEBIAN_FRONTEND=noninteractive
        
        apt-get update
        apt-get install -y software-properties-common
        apt-add-repository --yes --update ppa:ansible/ansible
        apt-get install -y ansible
        
        echo ">>> Ansible 安裝完畢！"
      fi

	  # 判斷 .ansible.cfg 是否已經存在
      if [ ! -f /home/vagrant/.ansible.cfg ]; then
        echo ">>> 找不到設定檔，建立全新的 .ansible.cfg 並關閉 Strict Host Key Checking..."
        echo -e "[defaults]\nhost_key_checking = False" > /home/vagrant/.ansible.cfg
        chown vagrant:vagrant /home/vagrant/.ansible.cfg
      else
        echo ">>> .ansible.cfg 已存在，跳過覆蓋動作，保留您的自訂設定。"
      fi
    SHELL

	
    end
end
