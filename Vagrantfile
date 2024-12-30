rh_user = ENV['RH_USER']
rh_pass = ENV['RH_PASS']

# Check if the environment variables are set.
if rh_user.nil? || rh_user.empty? || rh_pass.nil? || rh_pass.empty?
    raise 'Environment variables RH_USER and RH_PASS must be set.'
end

Vagrant.configure("2") do |config|
    # Ansible control server:
    config.vm.define "acs" do |acs|
        acs.vm.box = "generic/rhel8"
        acs.vm.hostname = "acs"

        # Provision script to register this VM with the RHEL subscription.
        acs.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
                set -euo pipefail

                echo "Registering the VM to Red Hat Subscription Management..."
                subscription-manager register --username #{rh_user} --password #{rh_pass} --auto-attach

                echo "Fixing missing nl language"
                yum install -y glibc-langpack-nl

                echo "Installing Ansible..."
                yum install -y ansible
                # NOTE: this will break once ansible will install a newer version of python3.
                python3.11 -m ensurepip
                # jmespath is needed by role ansible-ThoTeam.nexus3-oss:
                python3.11 -m pip install jmespath
                su -l vagrant -c "ansible-galaxy role install geerlingguy.java"
                su -l vagrant -c "ansible-galaxy role install ansible-ThoTeam.nexus3-oss"
            SHELL
        end

        # Make sure all sensitive info is only readable by user.
        acs.vm.synced_folder ".", "/vagrant", mount_options: ["dmode=700,fmode=600"]

        # Use the trigger feature to deregister when destroying the VM.
        acs.trigger.before :destroy do |trigger|
            trigger.name = "Deregistering VM..."
            trigger.run = { inline: "vagrant ssh acs -c 'sudo subscription-manager unregister'" }
        end

    end

    # Nexus server:
    config.vm.define "nexus" do |nexus|
        nexus.vm.box = "generic/rhel8"
        nexus.vm.hostname = "nexus"

        nexus.vm.network "private_network", ip: "192.168.14.34"

        # Provision script to register this VM with the RHEL subscription.
        nexus.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
                set -euo pipefail

                echo "Registering the VM to Red Hat Subscription Management..."
                subscription-manager register --username #{rh_user} --password #{rh_pass} --auto-attach
            SHELL
        end

        # Use the trigger feature to deregister when destroying the VM.
        nexus.trigger.before :destroy do |trigger|
            trigger.name = "Deregistering VM..."
            trigger.run = { inline: "vagrant ssh nexus -c 'sudo subscription-manager unregister'" }
        end
   end

   # Internet-less server:
    config.vm.define "nointernet" do |nointernet|
        nointernet.vm.box = "almalinux/8"
        nointernet.vm.hostname = "nointernet"

        nointernet.vm.network "private_network", ip: "192.168.14.35"

        nointernet.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
                set -euxo pipefail

                yum update -y
                
                yum install -y python3

                yum install -y firewalld
                systemctl enable --now firewalld
                
                # Allow HTTP(S) to 192.168 subnet:
                firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp --dport 80 -d 192.168.0.0/16 -j ACCEPT
                firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp --dport 443 -d 192.168.0.0/16 -j ACCEPT

                # Block HTTP(S) to everything else:
                firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport 80 -j REJECT
                firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport 443 -j REJECT

                firewall-cmd --reload
                firewall-cmd --list-all
            SHELL
        end
   end
end
