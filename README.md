# LoadBalancer using iptables

## 1. vagrant up

Use Vagrantfile provided

```sh
vagrant up
```

## 2. Setup simple nginx web application on application nodes

```sh
sudo apt install nginx -y
```

## 3. DNAT

```sh
sudo iptables \
  -A PREROUTING \
  -t nat \
  -p tcp \
  -d 192.168.50.2 \
  --dport 8080 \
  -j DNAT \
  --to-destination 192.168.50.11:80
```

## 4. Allow FORWARD for TCP requests
Enable kernet's ip_forward flag
```sh
sudo sed -i 's/\#net.ipv4.ip_forward/net.ipv4.ip_forward/' /etc/sysctl.d/99-sysctl.conf
```

```sh
sudo iptables \
  -A FORWARD \
  -p tcp \
  -d 192.168.50.11 \
  --dport 80 \
  -m state \
  --state NEW,ESTABLISHED,RELATED \
  -j ACCEPT
```

```sh
sudo iptables \
  -A FORWARD \
  -p tcp \
  -d 192.168.50.12 \
  --dport 80 \
  -m state \
  --state NEW,ESTABLISHED,RELATED \
  -j ACCEPT
```

## 5. SNAT

```sh
sudo iptables \
  -A POSTROUTING \
  -t nat \
  -p tcp \
  -d 192.168.50.11 \
  --dport 80 \
  -j SNAT \
  --to-source 192.168.50.2
```

## 6. Check connection from host machine

```sh
curl -XGET 192.168.50.2:8080 -I
```

## 7. Route requests to multiple application nodes

Prerequisites: delete all existing PREROUTING rules

```sh
sudo iptables -L -v -n -t nat

sudo iptables -D PREROUTING 1 -t nat
```

### Round robin

```sh
sudo iptables -A PREROUTING -t nat -p tcp -d 192.168.50.2 --dport 8080 \
         -m statistic --mode nth --every 2 --packet 0              \
         -j DNAT --to-destination 192.168.50.12:80
sudo iptables -A PREROUTING -t nat -p tcp -d 192.168.50.2 --dport 8080 \
         -j DNAT --to-destination 192.168.50.11:80
```

SNAT for node-2
```sh
sudo iptables \
  -A POSTROUTING \
  -t nat \
  -p tcp \
  -d 192.168.50.12 \
  --dport 80 \
  -j SNAT \
  --to-source 192.168.50.2
```

