# -*- mode: ruby -*-
# vi: set ft=ruby :

###################################################################
###################################################################
# Root provisioning
# (Vagrant runs these commands are run as the root user)
###################################################################
###################################################################
$root_provision_script = <<END_ROOT_PROVISIONING
echo "ROOT PROVISION SCRIPT STARTING (user="`whoami`", pwd="`pwd`")"

apt-get update

# BuildEM dependencies
apt-get install -y make
apt-get install -y cmake
apt-get install -y git
apt-get install -y mercurial
apt-get install -y build-essential
apt-get install -y gfortran

# Dependencies of ilastik not included in BuildEM
# (See BuildEM readme.)
apt-get install -y libxext-dev
apt-get install -y libgl1-mesa-dev
apt-get install -y libxt-dev
apt-get install -y libxml2-dev

# Since we'll use X11 for this VM, we must install these packages before building Qt.
# See http://qt-project.org/doc/qt-4.8/requirements-x11.html
# Note: Qt will *build* if some of these are omitted,
#        but you will encounter bugs at runtime.
apt-get install -y libfontconfig1-dev
apt-get install -y libxfixes-dev
apt-get install -y libxrender-dev
apt-get install -y libxcursor-dev
apt-get install -y libxrandr-dev
apt-get install -y libxinerama-dev

# XVFB allows us to run GUI tests in a virtual screen
apt-get install -y xvfb

# Fluxbox is a light-weight window manager, 
#  which we use during headless GUI tests (with xvfb)
apt-get install -y fluxbox

# The user can manipulate the "headless" screen 
#  by connecting to it via vnc.
apt-get install -y x11vnc

# Hudson compatibility
# Java is needed so this VM can run as a hudson slave
apt-get install -y openjdk-7-jre

# Create a workspace for Hudson to use
# (In Hudson, provide this path as the node's "Remote FS root")
HUDSON_REMOTE_FS_ROOT=/var/hudson
mkdir -p $HUDSON_REMOTE_FS_ROOT
chmod 777 $HUDSON_REMOTE_FS_ROOT

echo "ROOT PROVISION SCRIPT DONE"
END_ROOT_PROVISIONING
###################################################################
###################################################################


###################################################################
###################################################################
# Non-root provisioning
# (Vagrant runs these commands as the vagrant user)
###################################################################
###################################################################
$nonroot_provision_script = <<END_NONROOT_PROVISIONING
echo "NONROOT PROVISION SCRIPT STARTING (user="`whoami`", pwd="`pwd`")"
# Set up the build directory, but don't build
cd /home/vagrant
mkdir -p ilastik-build
cd ilastik-build
BUILDEM_DIR=`pwd`
if [ ! -d "$BUILDEM_DIR/ilastik-build-Linux" ]; then
    git clone https://github.com/ilastik/ilastik-build-Linux.git
else
    cd ilastik-build-Linux && git pull origin master && cd -
fi

mkdir -p build
cd /home/vagrant


##########################################################
# Edit .bashrc to activate BuildEM environment on login. #
##########################################################
# This is a little tricky.
# We mark the end of the .bashrc script to indicate the start of the 
#  auto-generated portion we are about to write.
# Then, we *delete* any previous auto-generated section using bash substitution.
# This way, even if the provision script is run many times for the same VM, the
#  .bashrc script doesn't end up with lots of duplicate auto-generated lines.
echo "" >> /home/vagrant/.bashrc
echo "" >> /home/vagrant/.bashrc
echo "#BEGIN_PROVISION_AUTOGENERATED_SECTION" >> /home/vagrant/.bashrc
bashrc_contents=$(cat .bashrc)
bashrc_without_autogen=${bashrc_contents%%#BEGIN_PROVISION_AUTOGENERATED_SECTION*}
echo "$bashrc_without_autogen" | head -n-2 > .bashrc
echo "" >> /home/vagrant/.bashrc
echo "#BEGIN_PROVISION_AUTOGENERATED_SECTION" >> /home/vagrant/.bashrc
echo "# Don't edit below this line!" >> /home/vagrant/.bashrc
echo "# It will be deleted the next time 'vagrant provision' is executed." >> /home/vagrant/.bashrc
echo "# Automatically activate the BuildEM environment, but ignore errors if it doesn't exist yet." >> /home/vagrant/.bashrc
echo "export BUILDEM_DIR=$BUILDEM_DIR" >> /home/vagrant/.bashrc
echo "source $BUILDEM_DIR/bin/setenv_ilastik_gui.sh 2> /dev/null" >> /home/vagrant/.bashrc

###################
# lazyflow config # 
###################
echo "Writing lazyflow config file"
mkdir -p ~/.lazyflow
echo "[verbosity]" > ~/.lazyflow/config
echo "deprecation_warnings = false" >> ~/.lazyflow/config

######################
# Download test data #
######################
echo "Downloading real-world test data"
TEST_DATA_DIR=/home/vagrant/real_test_data
if [ ! -d "$TEST_DATA_DIR" ]; then
    git clone http://github.com/ilastik/ilastik_testdata $TEST_DATA_DIR
else
    cd $TEST_DATA_DIR && git pull origin master && cd -
fi

#########################################################
# Write headless display activation/deactivation script #
#########################################################
cat <<END_HEADLESS_DISPLAY_CONTROL > headless_display_control.sh

XVFB=/usr/bin/Xvfb
XVFB_ARGS="\\$DISPLAY -ac -screen 0 1280x1024x24"
XVFB_PIDFILE=/tmp/headless_setup_xvfb.pid

FLUXBOX=/usr/bin/fluxbox
FLUXBOX_ARGS="-display \\$DISPLAY"
FLUXBOX_PIDFILE="/tmp/headless_setup_fluxbox.pid"

X11VNC=/usr/bin/x11vnc
X11VNC_ARGS="-rfbport 5900 -shared -forever"
X11VNC_PIDFILE="/tmp/headless_setup_x11vnc.pid"

case "\\$1" in
  start)
    # Check for errors...
    if [[ -z "\\$DISPLAY" ]]; then
        echo "DISPLAY environment variable is not set. Please set one, e.g. DISPLAY=:0"
        exit 1
    fi
    
    if \\`echo \\$DISPLAY | grep -q localhost\\`; then
        echo "Please set your DISPLAY environment variable for the virtual buffer, e.g. DISPLAY=:0"
        exit 1
    fi

    echo "Activating headless environment on current DISPLAY: \\$DISPLAY"
    echo "Starting virtual X frame buffer: Xvfb"
    /sbin/start-stop-daemon --start --pidfile \\$XVFB_PIDFILE --make-pidfile --background --exec \\$XVFB -- \\$XVFB_ARGS
    
    echo "Starting fluxbox window manager"
    /sbin/start-stop-daemon --start --pidfile \\$FLUXBOX_PIDFILE --make-pidfile --background --exec \\$FLUXBOX -- \\$FLUXBOX_ARGS

    echo "Starting x11vnc server on port 5900, which is forwarded to the host at 13900."
    /sbin/start-stop-daemon --start --pidfile \\$X11VNC_PIDFILE --make-pidfile --background --exec \\$X11VNC -- \\$X11VNC_ARGS
    echo "To view the guest's virtual frame buffer from your host machine, connect your vnc viewer to localhost:13900, DISPLAY=\\$DISPLAY"

    echo "Running the following processes:"    
    top -b -n1 | grep "\\(Xvfb\\)\\|\\(fluxbox\\)\\|\\(x11vnc\\)"
    ;;
  stop)
    echo "Stopping x11vnc server"
    /sbin/start-stop-daemon --stop --pidfile \\$X11VNC_PIDFILE
    rm -f \\$X11VNC_PIDFILE

    echo "Stopping fluxbox window manager"
    /sbin/start-stop-daemon --stop --pidfile \\$FLUXBOX_PIDFILE
    rm -f \\$FLUXBOX_PIDFILE

    echo "Stopping virtual X frame buffer: Xvfb"
    /sbin/start-stop-daemon --stop --pidfile \\$XVFB_PIDFILE
    rm -f \\$XVFB_PIDFILE
    ;;
  restart)
    \\$0 stop
    \\$0 start
    ;;
  *)

  echo "Usage: \\$0 {start|stop|restart}"
  exit 1
esac
exit 0
END_HEADLESS_DISPLAY_CONTROL

######################
# Write build script #
######################
cat <<END_BUILD_SCRIPT > build_ilastik.sh
#!/bin/bash
set -e
cd $BUILDEM_DIR/build
cmake ../ilastik-build-Linux -DBUILDEM_DIR=$BUILDEM_DIR -DILASTIK_VERSION=master
make "\\$@"
# make package
END_BUILD_SCRIPT
################

#####################
# Write test script #
#####################
cat <<END_TEST_SCRIPT > run_all_ilastik_tests.sh
#!/bin/bash

USE_XVFB=0
SKIP_ALL_GUI_TESTS=0
SKIP_RECORDED_GUI_TESTS=0

for arg in \\$@
do
    case \\$arg in
        "--use-xvfb")
            echo "Activating headless environment..."
            export DISPLAY=:0
            bash -e /home/vagrant/headless_display_control.sh start
            USE_XVFB=1
            ;;
        "--skip-gui-tests")
            echo "Skipping all GUI tests."
            SKIP_ALL_GUI_TESTS=1
            ;;
        "--skip-recorded-gui-tests")
            echo "Skipping recorded GUI tests."
            SKIP_RECORDED_GUI_TESTS=1
            ;;
    esac
done

# Run the test in a subshell (using the parens), 
#  so we can clean up afterwards if there was a failure.
(
    set -e # Exit on first failure.
    
    # Set up env
    export BUILDEM_DIR=$BUILDEM_DIR
    source \\$BUILDEM_DIR/bin/setenv_ilastik_gui.sh
    export PATH=\\$BUILDEM_DIR/bin:\\$PATH
    
    # Update repo to latest checkpoint
    # (This updates lazyflow, volumina, and ilastik)
    cd \\$BUILDEM_DIR/src/ilastik
    # Pull from ilastik github account, not janelia-flyem
    git remote add ilastik https://github.com/ilastik/ilastik-meta || : # no-op to avoid exit due to set -e
    git pull ilastik master
    git submodule update --init --recursive
    
    # Update all 3 repos to the latest commit, even though 
    #  that may not be the commit specified by the meta-repo.
    cd volumina && git checkout master && git pull origin master && git submodule update --init --recursive && cd -
    cd lazyflow && git checkout master && git pull origin master && git submodule update --init --recursive && cd -
    cd ilastik  && git checkout master && git pull origin master && git submodule update --init --recursive && cd -
    
    # Run tests
    echo "Running lazyflow tests...."
    cd lazyflow/tests
    nosetests .
    cd -
    
    echo "Running volumina tests...."
    cd volumina/tests
    nosetests .
    cd -
    
    echo "Generating synthetic test data"
    python ilastik/tests/bin/generate_test_data.py /home/vagrant/fake_test_data
    
    cd ilastik/tests
    echo "Running ilastik unit tests"
    SKIP_GUI_TESTS=\\$SKIP_ALL_GUI_TESTS ./run_each_unit_test.sh
    
    if [ \\$SKIP_ALL_GUI_TESTS -eq 0 -a \\$SKIP_RECORDED_GUI_TESTS -eq 0 ]; then
        echo "Running ilastik recorded GUI tests"
        ./run_recorded_tests.sh
    else
        echo "SKIPPING RECORDED TESTS"
    fi

    cd ../..
)
exit_code=\\$?

# Cleanup: Disable xvfb, then return with the subshell exit code.
if [[ \\$USE_XVFB -eq 1 ]]
then
    echo "Deactivating headless environment."
    bash -e /home/vagrant/headless_display_control.sh stop
fi
exit \\$exit_code

END_TEST_SCRIPT
#################

echo "NONROOT PROVISION SCRIPT DONE"
END_NONROOT_PROVISIONING
###################################################################
###################################################################


###################################################################
###################################################################
## Vagrant Config
###################################################################
###################################################################
Vagrant.configure("2") do |config|
  # Every Vagrant virtual environment requires a box to build off of.
  config.vm.box = "ilastikci"
  config.vm.provision :shell, :inline => $root_provision_script
  config.vm.provision :shell, :privileged => false, :inline => $nonroot_provision_script
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"
  config.vm.hostname = "ilastikci"

  # Enable x11 forwarding (as if using ssh -X)
  config.ssh.forward_x11 = true

  # VirtualBox settings
  config.vm.provider :virtualbox do |vb|
    # Don't boot with headless mode
    #vb.gui = true

     # Use VBoxManage to customize the VM.
     # For more options, check the help message for the VBoxManage command
     vb.customize ["modifyvm", :id,
     			   "--memory", "2048",
     			   "--cpus", "4",
     			   "--cpuexecutioncap", "100" ]
  end

  # ADDITIONAL OPTIONS:

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network :forwarded_port, guest: 80, host: 8080
  config.vm.network :forwarded_port, guest: 22, host: 8022
  config.vm.network :forwarded_port, guest: 5900, host: 13900

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network :private_network, ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network :public_network, :bridge => 'eth0'

end
