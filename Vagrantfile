Vagrant.configure("2") do |config|
 
  #We announce the variables. Let's assign them the value of environment variables
  project_dir = ENV['PROJECT_DIR']
  db_user = ENV['DB_USER']
  db_pass = ENV['DB_PASS']
  db_name = ENV['DB_NAME']
  db_host = ENV['DB_HOST']
  mysql_url = "jdbc:mysql://#{db_host}/#{db_name}"
  app_user = ENV['APP_USER']
  puts "#{db_host}"
  
#I describe the first virtual machine. The database should start first
  config.vm.define "DB_VM" do |db_vm|
    db_vm.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"  # Memory size in megabytes
      vb.cpus = 2        # The number of processor cores
    end
    db_vm.vm.box = "ubuntu/focal64"
    db_vm.vm.network "private_network", ip: "#{db_host}"
    db_vm.vm.hostname = "db-vm"
    
    db_vm.vm.provision "shell", inline: <<-SHELL
      #install mysql server. listens on a private network
      sudo apt-get update
      sudo apt-get install -y mysql-server
      sudo sed -i "s/^bind-address.*/bind-address = #{db_host}/" /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo systemctl restart mysql
      
      #create a user and database
      sudo mysql -e "CREATE USER '#{db_user}'@'10.11.11.11' IDENTIFIED BY '#{db_pass}';"
      sudo mysql -e "CREATE DATABASE #{db_name};"
      sudo mysql -e "GRANT ALL PRIVILEGES ON #{db_name}.* TO '#{db_user}'@'10.11.11.11';"
            
    SHELL
  end

  config.vm.define "APP_VM" do |app_vm|
    app_vm.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"  # Memory size in megabytes
      vb.cpus = 4        # The number of processor cores
    end
    project_dir = ENV['project_dir']
    app_vm.vm.box = "ubuntu/focal64"
    app_vm.vm.network "private_network", ip: "10.11.11.11"
    app_vm.vm.hostname = "app-vm"
    app_vm.vm.provision "shell", inline: <<-SHELL
      #install java and git
      sudo apt-get update
      sudo apt-get install -y default-jdk 
      sudo apt-get install -y git
      sudo echo "export MYSQL_URL=#{mysql_url}" >> /etc/environment
      sudo echo "export MYSQL_USER=#{db_user}" >> /etc/environment
      sudo echo "export MYSQL_PASS=#{db_pass}" >> /etc/environment
      
      #create working folder
      mkdir #{project_dir}
      cd #{project_dir}
      
      #clon repo with ssh key
      ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
      ssh-agent bash -c 'ssh-add /vagrant/id_rsa; git clone git@gitlab.com:dan-it/groups/devops3.git'

      #build jar file
      cd #{project_dir}/devops3/StepProjects/PetClinic
      chmod +x mvnw
      ./mvnw package

      #create user and run  the application
      sudo useradd -m -d /home/#{app_user} #{app_user}
      cp target/*.jar /home/#{app_user} 
      cd /home/#{app_user}
      sudo chown #{app_user}:#{app_user} *.jar
      sudo -u #{app_user} nohup java -Dspring.profiles.active=mysql  -jar *.jar &
      
    SHELL
    app_vm.vm.network "forwarded_port", guest: 8080, host: 8080, auto_correct: true
  end
end

