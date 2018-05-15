
Docker Container Network에 대한 이해
==================================

## docker0와 container network의 구조 ##
 
**NOTE**

* 아래 그림은 docker의 network 구조를 간단히 도식화한 것이다.

* 여기서는 docker를 설치하면 가장 먼저 볼 수 있는 docker0 interface와 container network에 대해 알아본다.

 ![image](https://user-images.githubusercontent.com/20153890/40031808-49f6a9f2-582c-11e8-9c51-052ad4ddcbcf.png)

## 1. docker0 interface ##
**Docker host를 설치한 후 host의 network interface를 보면, docker 0 이라는 interface를 볼 수 있다.**

>	- $ifconfig 


![image](https://user-images.githubusercontent.com/20153890/40032017-392a7c56-582d-11e8-956c-5dea8308f525.png)

이 docker0 interface의 특징은, 
> - * IP는 자동으로 172.17.0.1로 배정된다. 
> - * IP는 DHCP로 자동할당이 되는 것이 아니고, docker 내부 로직에 따라 자동할당 된다.
> - * 이 docker0은 일반적은 interface가 아니고, virtual ethernet bridge이다.

즉 docker0은 Container가 통신하기 위한 virtual bridge라고 볼 수 있다.

하나의 Container가 생성시, 이 bridge에 container의 interface가 하나씩 binding되는 형태이다.
그리고 container가 running될 때 마다 vethXXXX라는 이름의 interface가 attach되는 형태이다.

결론적으로 container가 외부로 통신할 때는 무조건 docker0 interface를 지나야 한다.

또한 docker0의 ip는 자동으로 172.17.0.1로 설정되고, subnet mask는 255.255.0.0 (172.17.0.0/16) 으로 설정된다.
이 subnet 정보는 container가 생성될 때마다 container가 할당받게 될 IP의 range를 결정하게 된다.

**즉 모든 container는 172.17.XX.YY 대역에서 IP를 하나씩 할당받게 된다.**

 

## 2. container Network의 구조 ##

docker는 host에 container를 생성하게 되었을때, 각 conatiner는 격리된 공간을 할당받는다.

그렇다면 이 격리된 container는 어떻게 외부(또 다른 container로) 와 통신을 할까?
  
![image](https://user-images.githubusercontent.com/20153890/40032150-c9f5701a-582d-11e8-8813-1c292cba2e71.png)


먼저 Container가 생성되면, 해당 container에는 pair (peer) interface라고 하는 한 쌍의 interface가 생성된다.
이 pair interface는 두 interface가 한쌍으로 구성되고, 마치 direct로 연결한 두 대의 PC처럼 packet을 주고받는다.

즉 container 생성시, pair interface의 한쪽은 container내부에 할당되고, eh0이라는 이름으로 할당되고,
다른 한쪽은 vethXXX라는 이름으로 docker0 bridge에 binding된다.

> - $ip link
 

Container를 하나 올린 상태에서 link를 확인해보면, running중인 container는 vethXXX라는 이름으로 docker0 bridge에 연결되어 있는 것을 확인 할 수 있다.
 

그럼 container내부에 할당된 eth0 interface는 어떻게 확인할까?
이는 해당 namespace에서만 보이도록 격리되어 있으므로 running중인 container 내부에서 확인해야 한다.

	$docker exec {container id} ifconfig eth0

container안에 외부 통신을 위한 eth0 interface가 있는 것을 볼 수 있고, 
172.17.0.2라는 IP가 할당되었으며, netmask도 255.255.0.0으로 설정되어져 있는 것을 볼 수 있다.

그리고 이 container의 gateway는 docker0에 설정된 ip인 172.17.0.1로 되어져 있다.

	$docker exec {container id} route
 
Container 내부의 모든 packet은 default인 172.17.0.1(docker0의 ip)로 가게 된다.

지금까지 docker의 bridge모드에 대한 설명을 해보았다.
brige모드는 docker network의 default설정이자, 가장 많이 쓰이는 방식이다.


