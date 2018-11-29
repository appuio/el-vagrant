# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'net/ssh'

required_plugins = %w(
  vagrant-hostmanager
  vagrant-triggers
  vagrant-timezone
)

def install_plugins(plugins)
  needs_restart = false
  plugins.each do |plugin|
    unless Vagrant.has_plugin? plugin
      system "vagrant plugin install #{plugin}"
      needs_restart = true
    end
  end

  if needs_restart
    exec "vagrant #{ARGV.join' '}"
  end
end

def vagrant_dir()
  dir = Pathname.new(Dir.pwd)
  loop do
    return dir if (dir + 'Vagrantfile').exist?
    return nil if dir.root?
    dir = dir.parent
  end
end

install_plugins(required_plugins)
vagrant_dir = vagrant_dir()

# Generate a ssh key for mutual machine login
if vagrant_dir and not File.exist?("#{vagrant_dir}/.ssh/id_rsa")
  FileUtils.mkdir_p "#{vagrant_dir}/.ssh", :mode => 0700

  key = OpenSSL::PKey::RSA.new 2048
  type = key.public_key.ssh_type
  data = [ key.public_key.to_blob ].pack('m0')

  File.open("#{vagrant_dir}/.ssh/id_rsa", 'w', 0600) do |file|
    file.write key.to_pem
  end

  File.open("#{vagrant_dir}/.ssh/id_rsa.pub", 'w', 0600) do |file|
    file.puts "#{type} #{data} ansible"
  end
end

if Vagrant::Util::Platform.windows?
  hostname=`hostname`.chomp
else
  hostname=`hostname -s`.chomp
end

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"

  # vagrant-timezone: Set machine time zone based on host time zone
  config.timezone.value = :host

  # vagrant-hostmanager: Configure /etc/hosts in machines so that they can look up each other
  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  # vagrant-triggers: Try to unregister machine from Red Hat before it's destroyed when subscription-manager is installed
  config.trigger.before :destroy do
    case @machine.state.id
    when :not_created
    when :running
      run_remote "command -v subscription-manager >/dev/null && subscription-manager unregister", :force => true
    else
      info "Can't unregister VM when it's not running! Please start/resume VM before destroying it."
      exit
    end
  end

  # We handle subscriptions ourselves, disable vagrant-registration plugin
  if Vagrant.has_plugin?('vagrant-registration')
    config.registration.skip = true
  end

  config.vm.provider :libvirt do |libvirt|
    # Avoid "Call to virDomainCreateWithFlags failed: unsupported configuration: host doesn't support invariant TSC" error when using snapshots
    libvirt.cpu_mode = 'host-passthrough'   
  end

  # Distribute previously created ssh keys so machines can log in to each other, e.g. for Ansible
  # Register machine to Red Hat if subscription-manager is installed, using the name "vagrant-<hostname>-<machinename>"
  config.vm.provision "shell", name: "el-vagrant", inline: <<-SHELL
    # workaround https://github.com/devopsgroup-io/vagrant-hostmanager/issues/203
    sed -i "/^127.0.0.1.*$(hostname).*/d" /etc/hosts

    for user in root vagrant; do
      home=$(getent passwd ${user} | cut -d: -f6)
      if [ ! -e ${home}/.ssh/id_rsa ]; then
        mkdir -p ${home}/.ssh
        cp -p /vagrant/.ssh/id_rsa ${home}/.ssh
        cat /vagrant/.ssh/id_rsa.pub >>${home}/.ssh/authorized_keys
        chown -R ${user}:${user} ${home}/.ssh
      fi
    done

    if [ "#{$rhsm_username}" ] && command -v subscription-manager >/dev/null && ! subscription-manager identity >/dev/null 2>&1; then
      subscription-manager register --username="#{$rhsm_username}" --password="#{$rhsm_password}" --name="vagrant-#{hostname}-$(hostname)"
      subscription-manager attach #{$rhsm_pool ? "--pool=#{$rhsm_pool}" : ''}
      subscription-manager repos --enable="rhel-7-server-rpms" --enable="rhel-7-server-extras-rpms"
    fi

  SHELL

end
