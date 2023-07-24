# System Setup

ROS 2 is offically supported on Ubuntu Jammy (22.04). This guide assumes that Ubuntu Jammy (22.04) is installed. It also assumes that roskafka and the Kafka Streams pipeline are installed on the same machine. Kafka uses a lot of resources, so it is recommended to use a machine with at least 16 GB of RAM and 25 GB disk space.

The Turtlebots and the machine need to be on the same WiFi network. It is strongly suggested to keep the network infrastructure as simple as possible: ROS 2 uses multicast for its discovery service, and the Kafka ports need to be exposed via port forwarding (9092, 8081). Installing Ubuntu in a virtual machine and using bridge networking can be tricky because the Docker images for Kafka also rely on bridge networking. When using an isolated network (i.e., without Internet access), time synchronization may become an issue because NTP is not available.

## Initial Steps

- Update system

        sudo apt update
        sudo apt upgrade

- Confirm that locale is set to `en_US.UTF-8`

        localectl

- Confirm that NTP is active

        timedatectl


## Install Java 17

    sudo apt install openjdk-17-jdk
    sudo update-java-alternatives -s java-1.17.0-openjdk-amd64


## Install Python dependencies

    sudo apt install python3-pip
    pip3 install confluent-kafka # tested with version 2.1.1
    pip3 install fastavro # tested with version 1.7.4


## Install ROS

Adapted from https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html

    sudo apt install software-properties-common
    sudo add-apt-repository universe
    sudo apt install curl -y
    sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
    sudo apt update
    sudo apt upgrade
    sudo apt install ros-humble-desktop ros-dev-tools


## Install Docker and Docker Compose

Adapted from https://docs.docker.com/engine/install/ubuntu/

    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo usermod -aG docker $USER

Log out and back in for the group membership changes to take place.

## Install and run Kafka

    # Clone repo containing docker-compose.yml
    git clone https://gitlab.informatik.hs-furtwangen.de/ss23-forschungsprojekt-7/kafka-streams-turtlebots4.git
    cd kafka-streams-turtlebots4

    # Change the advertised IP/URL to the current IP of the VM.
    # This one must be externally reachable
    export HOST_IP=<your_ip>

    # Start Kafka
    sudo docker compose up -d


## Source ROS 2

    source /opt/ros/humble/setup.bash


## Install and run roskafka

    mkdir -p ~/ros2_ws/src
    cd ~/ros2_ws
    git clone https://gitlab.informatik.hs-furtwangen.de/ss23-forschungsprojekt-7/roskafka_interfaces.git src/roskafka_interfaces
    git clone https://gitlab.informatik.hs-furtwangen.de/ss23-forschungsprojekt-7/roskafka.git src/roskafka
    PYTHONWARNINGS=ignore:::setuptools.command.install colcon build

Open a new terminal: Always source a workspace from a different terminal than the one you used `colcon build`.

    source /opt/ros/humble/setup.bash
    source ~/ros2_ws/install/setup.bash

    export BOOTSTRAP_SERVERS=<broker_server>
    export SCHEMA_REGISTRY=<schema_registry>

    ros2 launch roskafka roskafka.launch.yaml


## Install and run kafka-streams app

    # Clone repo
    git clone https://gitlab.informatik.hs-furtwangen.de/ss23-forschungsprojekt-7/kafka-streams-turtlebots4 ~/kafka-streams
    cd kafka-streams

    # Download Avro schemas
    ./mvnw io.confluent:kafka-schema-registry-maven-plugin:7.4.0:download
    
    # Build Java classes from Avro schemas
    ./mvnw compile
    
    # Run application
    ./mvnw compile quarkus:dev
