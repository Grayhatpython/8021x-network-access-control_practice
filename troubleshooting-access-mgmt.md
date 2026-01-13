### 토폴로지

<img width="755" height="705" alt="스크린샷 2026-01-13 213744" src="https://github.com/user-attachments/assets/1340c29e-d828-4af7-a50e-bbe927beede9" />


## 🧩 Troubleshooting Log — Admin PC(VLAN300) → Access Switch Mgmt(VLAN110) Ping/SSH 실패

### 📌 상황 요약
- Admin PC(리눅스/QEMU): `10.0.30.5/24` (VLAN300)
- Backbone Switch(BSW):  
  - `Vlan300 SVI = 10.0.30.1/24`  
  - `Vlan110 SVI = 10.0.11.1/24`
- Access Switch(ASW1):  
  - `Vlan110(SVI) = 10.0.11.11/24` (관리 IP)
  - (L2 스위치로 사용)
- 목표: **Admin PC에서 ASW1 관리 IP(10.0.11.11)로 Ping/SSH 접속**

---

### 1) 증상
Admin PC에서 아래를 실행하면,

ping -c 5 10.0.11.11
결과는 항상:
5 packet transmitted, 0 received, 100% packet loss


처음엔 “라우팅이 안 됐나?” 싶어서 전체 경로를 하나씩 뜯어보기로 했다.

2) PC 쪽(10.0.30.5)부터 확인 — 기본 게이트웨이/라우팅 정상

<img width="1036" height="851" alt="스크린샷 2026-01-13 160150" src="https://github.com/user-attachments/assets/614cffef-1aba-4f27-b8a7-6e18e2c24b7c" />

PC netplan 설정(요약):
IP: 10.0.30.5/24
Default GW: 10.0.30.1

ip route
default via 10.0.30.1 dev ens3 proto static
10.0.30.0/24 dev ens3 proto kernel scope link src 10.0.30.5

GW 핑 테스트:

ping -c 3 10.0.30.1
# 3 transmitted, 3 received (정상)
✅ 결론: PC 설정/게이트웨이는 문제 없음

3) BSW 라우팅 확인 — VLAN300은 Connected로 정상
BSW에서 PC 대역 라우트 확인:

BSW#show ip route 10.0.30.5
Routing entry for 10.0.30.0/24
  Known via "connected"
  * directly connected, via Vlan300

SVI 상태도 확인:

BSW#show ip interface brief | include Vlan
Vlan110 10.0.11.1 up/up
Vlan300 10.0.30.1 up/up
...
✅ 결론: BSW의 SVI/라우팅 상태는 정상

4) ASW1 관리 VLAN(L2 연결) 확인 — VLAN110은 살아있음
ASW1은 L2 스위치로 쓰고 있었고(ip routing 없음), 관리 SVI는 VLAN110에만 존재.

ASW1#show running-config | include ip routing
# 출력 없음 (ip routing 비활성)
VLAN110 SVI 상태:

ASW1#show ip interface brief
Vlan110 10.0.11.11 up/up
그리고 “BSW(10.0.11.1)까지 L2로 닿는지”를 ARP로 확인:

ASW1#show ip arp | include 10.0.11.1
Internet 10.0.11.1  ...  ARPA  Vlan110
트렁크 확인:

ASW1#show interfaces trunk
Gi1/1 trunking, allowed vlan 1-4094, native vlan 999
✅ 결론: ASW1 ↔ BSW 사이 VLAN110 L2/ARP/트렁크는 정상

5) “진짜로 패킷이 어디서 끊기는지” tcpdump로 증거 잡기
이제 설정을 더 보는 것보다, 실제 패킷이 왕복하는지가 중요했다.

Admin PC에서:
sudo tcpdump -ni ens3 icmp or arp
PC에서 ping 10.0.11.11을 날리면 tcpdump에는 아래가 보였다:
ARP로 게이트웨이(10.0.30.1)는 정상 학습
ICMP echo request(10.0.30.5 → 10.0.11.11)는 찍힘
그런데 echo reply(10.0.11.11 → 10.0.30.5)는 끝까지 안 찍힘

<img width="955" height="841" alt="스크린샷 2026-01-13 203149" src="https://github.com/user-attachments/assets/b814d434-42cf-4796-8d5a-ee27f81c738f" />

# echo reply 없음
✅ 결론: PC는 요청을 보내고 있는데, 응답이 생성/전달되지 않음

6) 중간에 계속 보이던 “8.8.8.8 unreachable”는 노이즈였다
tcpdump에 이런 로그가 계속 같이 섞여서 처음엔 더 헷갈렸다.

<img width="1038" height="1118" alt="스크린샷 2026-01-13 161340" src="https://github.com/user-attachments/assets/0eb56cee-0cef-4cd3-9850-2ddc1342c316" />


IP 10.0.30.1 > 10.0.30.5: ICMP host 8.8.8.8 unreachable
IP 10.0.30.1 > 10.0.30.5: ICMP host 8.8.4.4 unreachable
PC의 netplan DNS가 8.8.8.8 / 8.8.4.4로 되어 있고,
랩망에서 외부로 나가는 경로가 없으니 BSW(10.0.30.1)가 unreachable을 돌려주는 상황이었다.
이번 “ASW1 관리 IP ping/ssh 실패”의 직접 원인은 아니고 디버깅 노이즈였다. -> 설정안되어있음

7) Packet Tracer에서는 되는데, EVE(vIOS-L2)에서는 안 되는 이유(결론)
같은 설정을 Packet Tracer에서 하면 다른 서브넷(VLAN300)에서도
Access Switch 관리 SVI(VLAN110)에 ping이 되는 경우가 있었다.

하지만 EVE-NG의 vIOS-L2에서는:
Access 스위치의 관리 SVI(컨트롤 플레인) 를
사용자 VLAN(다른 서브넷)에서 직접 ICMP/SSH로 접근하는 것을 제한하거나

환경/정책/구현 차이로 인해 응답이 생성되지 않는 동작을 보였다.

즉, “라우팅이 안 돼서”가 아니라,
관리 VLAN과 사용자 VLAN을 분리한 설계에서는 원래 ‘관리 경로’를 따로 만들어야 한다는 쪽이 정답에 가까웠다.
