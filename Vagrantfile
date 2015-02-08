unless Vagrant.has_plugin?("vagrant-reload")
  raise "プラグイン vagrant-reload がインストールされていません。"
end

Vagrant.configure(2) do |config|
  config.ssh.insert_key = false

  config.vm.define "piyo" do |c|
    c.vm.box = "chef/centos-7.0"
    c.vm.provision "ansible" do |ansible|
      ansible.playbook = "provision1.yml"
    end
    c.vm.provision "reload"
    c.vm.provision "ansible" do |ansible|
      ansible.playbook = "provision2.yml"
    end
    c.vm.provision "reload"
  end

end
