### Chức năng:
So sánh tìm kiếm mẫu các netflow đã biết với luồng netflow đo được thực tế. 

Trong đó, các netflow mẫu được biểu thị vào dưới dạng vector request, vector response đã đo từ trước. Nhập vào từ file configuration.yml - chứ không thể đưa trực tiếp vào ứng dụng

Làm sao ra được trọng số của mỗi vector? application này không thấy nói đến.

Mã nguồn tại:

https://github.com/CSIRT-MU/Stream4Flow/blob/master/applications/detection/pattern_finder/spark/pattern_finder.py

Ví dụ một file configuration cho detect ssh_authent_attack :

https://github.com/CSIRT-MU/Stream4Flow/blob/master/applications/detection/pattern_finder/additional_data/SSH_authentication_attack_detection/configuration.yml

### Stream input : 

```
{"@type": "ipfix.entry", "ipfix.flowStartMilliseconds": 1537599474880, "ipfix.flowEndMilliseconds": 1537599485694, "ipfix.forwardingStatus": 0, "ipfix.tcpControlBits": 27, "ipfix.protocolIdentifier": 6, "ipfix.ipClassOfService": 0,
"ipfix.sourceTransportPort": 34306, "ipfix.destinationTransportPort": 22, "ipfix.icmpTypeCodeIPv4": 0, "ipfix.sourceIPv4Address": "240.0.1.4", "ipfix.destinationIPv4Address": "240.125.0.2", "ipfix.packetDeltaCount": 16, "ipfix.octetDeltaCount": 1973, "ipfix.ingressInterface": 0, "ipfix.egressInterface": 0, "ipfix.bgpSourceAsNumber": 0, "ipfix.bgpDestinationAsNumber": 0}
```

### Stream Output: 

```
{"src_ip":"240.0.1.4","closest_patterns":["hydra","ncrack-1","ncrack-2"],"data_array":[{"distribution":[126,2,0,0,0,0],"name":"hydra"},{"distribution":[126,2,0,0,0,0],"name":"ncrack-1"},{"distribution":[126,2,0,0,0,0],"name":"ncrack-2"},{"distribution":[0,86,0,0,0,0],"name":"medusa"}],"dst_ip":"240.125.0.2","configuration":"SSH Brute-force Attack Detection","@type":"pattern_finder"}
```
### File configuration.yml: 

File có nhiệm vụ đưa các tham số trong khi chạy mỗi hàm.

Ví dụ file configuration.yml :
```
# -------------------------------------- Common Application Settings -------------------------------------- #
configuration:
    name: SSH Brute-force Attack Detection
    window: 300
    slice: 5


# -------------------------------------- Input Data Filter  ----------------------------------------------- #
filter:
    - element_names:
          - ipfix.sourceIPv4Address
          - ipfix.destinationIPv4Address
      type: exists
    - element_names:
          - ipfix.sourceTransportPort
          - ipfix.destinationTransportPort
      type: int
      values:
          - 22
    - element_names:
          - ipfix.protocolIdentifier
      type: int
      values:
          - 6

# -------------------------------------- Vector Definition ------------------------------------------------ #
vectors:
    key:
        type: biflow
        elements:
            src_ip: ipfix.sourceIPv4Address
            dst_ip: ipfix.destinationIPv4Address
            src_port: ipfix.sourceTransportPort
            dst_port: ipfix.destinationTransportPort
            flow_start: ipfix.flowStartMilliseconds
        time_difference: 500
    values:
        - type: element
          element: ipfix.packetDeltaCount
        - type: element
          element: ipfix.octetDeltaCount
        - type: direct
          value: 1
        - type: operation
          operator: sub
          elements:
              - ipfix.flowEndMilliseconds
              - ipfix.flowStartMilliseconds


# -------------------------------------- Additional Output Fields ----------------------------------------- #
output:
    - name: src_ip
      element: ipfix.sourceIPv4Address
      type: request
    - name: dst_ip
      element: ipfix.destinationIPv4Address
      type: request


# -------------------------------------- Distance Function and Pattern Definition ------------------------- #
distance:
    distance_module: biflow_quadratic_form
    patterns:          
        - name: hydra
          request: [16, 1973, 11959.5]
          response: [25, 3171, 11959.5]
        - name: medusa
          request: [18, 2528, 6079]
          response: [25, 3715, 6079]
        - name: ncrack-1
          request: [13, 2860, 2549.5]
          response: [14, 2103, 2548.5]
        - name: ncrack-2
          request: [16, 3340, 10050]
          response: [21, 2675, 10048]
    distribution:
        default:
            intervals: [0, 2, 3, 4, 5, 7]
            weights: [3, 2, 1, 1, 2, 3]
            limit: 7

```
Việc giải thích file sẽ đi kèm khi giải thích thuật toán.
<br/>

### Thuật toán, cài đặt.

Bước 1: Lọc các bản ghi đáp ứng điều kiện theo file configuration đã chỉ ra:

ví dụ ở file configuration ở trên, phần Filter:
```
# -------------------------------------- Input Data Filter  ----------------------------------------------- #
filter:
    - element_names:
          - ipfix.sourceIPv4Address
          - ipfix.destinationIPv4Address
      type: exists
    - element_names:
          - ipfix.sourceTransportPort
          - ipfix.destinationTransportPort
      type: int
      values:
          - 22
    - element_names:
          - ipfix.protocolIdentifier
      type: int
      values:
          - 6
```
Đã được process dịch ra thành các điều kiện:

&nbsp;   &nbsp;- có cả source, dest IP (type exits) 

&nbsp;   &nbsp;- source Port hoặc dest Port phải là 22 (type int)

&nbsp;   &nbsp;- protocolIdentifier là 6 (tcp) (type int)

Ta có thể thêm vào trường filter với các type bao gồm:

 &nbsp;     &nbsp; ge:  giá trị của element_names phải lớn hơn value
 
 &nbsp;     &nbsp; nin: giá trị của element_names không thuộc trong values
 
 &nbsp;     &nbsp; lt:  giá trị của element_names phải nhỏ hơn value
  
 &nbsp;     &nbsp; le:  giá trị của element_names  phải nhỏ hơn hoặc bằng value
  
 &nbsp;     &nbsp; eq:  giá trị của element_names phải bằng value
  
 &nbsp;     &nbsp; ne:  giá trị của element_names phải khác value
   
 &nbsp;     &nbsp; ge:  giá trị của element_names lớn hơn hoặc bằng value
   
 &nbsp;     &nbsp; gt:  giá trị của element_names lơn hơn value
   

Bước 2: Tạo ra luồng các "kết nối", mỗi bản ghi là một cặp key-value như sau:


```
_________key________|_______________value______________________________________
 (ip1-ip2)          | {src_ip , 'dst_ip', {vector-request[]; vector-response[]}
-------------------------------------------------------------------------------
```
vd :
```
('240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11293.0], 'response': [25, 3171, 1, 11293.0]}})
```

Với giá trị vector request, respone được khai báo trong file configuration:

vd file: 
```
#-------------------------------------- Vector Definition ------------------------------------------------ #
vectors:
    key:
        type: biflow
        elements:
            src_ip: ipfix.sourceIPv4Address
            dst_ip: ipfix.destinationIPv4Address
            src_port: ipfix.sourceTransportPort
            dst_port: ipfix.destinationTransportPort
            flow_start: ipfix.flowStartMilliseconds
        time_difference: 500
    values:
        - type: element
          element: ipfix.packetDeltaCount
        - type: element
          element: ipfix.octetDeltaCount
        - type: direct
          value: 1
        - type: operation
          operator: sub
          elements:
              - ipfix.flowEndMilliseconds
              - ipfix.flowStartMilliseconds
```
Ta có:
&nbsp;     &nbsp;-Trường value của vectors: xác định danh sách các trường trong vector. 
&nbsp;     &nbsp;vd:như file trên, vector sẽ có các trường [packetDeltaCount; octetDeltaCount; 1; flow_end - flow_start]
&nbsp;     &nbsp;-Trường key của vectors: xác định xem các bản ghi như thế nào sẽ được coi như cùng thuộc một "kết nối"
&nbsp;     &nbsp;vd:như file trên, 1 kết nối khi các bản ghi có: cùng src-ip,src-port, dst-ip, dst-port; time_difference giữa flow start 2 bản ghi không quá 500ms

Thực hiện lọc luồng key-value mới tạo. thành luồng mới theo tiêu chí: 
Nhóm các bản ghi cùng 1 key:  value:
    - src_ip và flow_start khác nhau
    - giữa 2 bản ghi bất kì trong luồng không có |flow_start1 -flowstart2| < time_difference (time_difference từ file configuration.yml)
    - lấy bản ghi có source-port nhỏ nhất làm đại diện

Ý nghĩa bước này: với mỗi kết nối 
Map tạo một luồng mới với key-value:

```
_________key________________________|_______________value________________________
 (src_port:src_ip-dst_port:dst_ip)  | {flow_start, src_port, src_ip, vector[]}
---------------------------------------------------------------------------------
```

vd:
```
('240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11293.0], 'response': [25, 3171, 1, 11293.0]}})
('240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11002.0], 'response': [25, 3171, 1, 11002.0]}})
```
* Bước 8: khoảng cách so với các vector cho trước 
8.1

```
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11293.0], 'response': [25, 3171, 1, 11293.0]}, 'distances': {'hydra': 1.414095312148389, 'ncrack-1': 1.7315978357906066, 'ncrack-2': 1.7320508075688772, 'medusa': 2.23606797749979}})
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11002.0], 'response': [25, 3171, 1, 11002.0]}, 'distances': {'hydra': 1.414095312148389, 'ncrack-1': 1.7315978357906066, 'ncrack-2': 1.7320508075688772, 'medusa': 2.23606797749979}})
```
8.2
flows_distribution:
```
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11293.0], 'response': [25, 3171, 1, 11293.0]}, 'distributions': {'hydra': [3, 0, 0, 0, 0, 0], 'ncrack-1': [3, 0, 0, 0, 0, 0], 'medusa': [0, 2, 0, 0, 0, 0], 'ncrack-2': [3, 0, 0, 0, 0, 0]}, 'distances': {'hydra': 1.414095312148389, 'ncrack-1': 1.7315978357906066, 'ncrack-2': 1.7320508075688772, 'medusa': 2.23606797749979}})
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'vector': {'request': [16, 1973, 1, 11002.0], 'response': [25, 3171, 1, 11002.0]}, 'distributions': {'hydra': [3, 0, 0, 0, 0, 0], 'ncrack-1': [3, 0, 0, 0, 0, 0], 'medusa': [0, 2, 0, 0, 0, 0], 'ncrack-2': [3, 0, 0, 0, 0, 0]}, 'distances': {'hydra': 1.414095312148389, 'ncrack-1': 1.7315978357906066, 'ncrack-2': 1.7320508075688772, 'medusa': 2.23606797749979}})
```
8.3
distributions_sum:
```
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'distributions': {'hydra': [21, 0, 0, 0, 0, 0], 'ncrack-1': [21, 0, 0, 0, 0, 0], 'medusa': [0, 14, 0, 0, 0, 0], 'ncrack-2': [21, 0, 0, 0, 0, 0]}})
```

8.4
anomalies

```
(u'240.0.1.4-240.125.0.2', {'output': {'src_ip': u'240.0.1.4', 'dst_ip': u'240.125.0.2'}, 'distributions': {'hydra': [21, 0, 0, 0, 0, 0], 'ncrack-1': [21, 0, 0, 0, 0, 0], 'medusa': [0, 14, 0, 0, 0, 0], 'ncrack-2': [21, 0, 0, 0, 0, 0]}})
```

