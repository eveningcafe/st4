## [1. Tài liệu] (#Tài liệu)
## [2. Tính năng] (#Tính năng)
## [3. Các thành phần trong soucre code] (#Các thành phần trong soucre code)
## [4. Cách thức hoạt động] (#Cách thức hoạt động)
## [5. Mô hình cài đặt- kết hợp sahara] (#Mô hình cài đặt- kết hợp sahara)
## [6. Chạy các application] (#Chạy các application)
## <a name="Tài liệu"></a> 1. Tài liệu

### Doc
https://is.muni.cz/th/cuoo5/BP_Ondrej_Zoder.pdf

## <a name="Tính năng"></a> 2. Tính năng

Stream4Flow là một *framework* cho việc phân tích IP flow data một cách real-time, phân tán, hiệu năng cao. Là sự kết hợp các tool thành phần về flow-colection (IPFIXCol), dataprocessing (kafka, spark), storage-search-visualized(ELK-Stack)

Các chức năng được giới thiệu :

Với đầu vào là từ các gói netflow, ipfix nhận được, các chương trình có thể chạy:

*- Statistics:* 
* Protocol statistics: Thống kê theo giao thức (tcp,udp, other..) số lượng  flows, packets, bytes ; 10 s đếm một lần
* Host statistics: Thống kê, lọc theo host, network : số lượng flows, packets , bytes; số port đích, số kết nối,..
* DNS statistics: Thống kê số request, respone, địa chỉ các dns server mà một host cho trước gửi đi.

*- Detection :* 

* DDOS detection: phát hiện ddos dựa trên mật độ vào ra bất thường các packet của một host/network cho trước.
* DNS Servers Usage: phát hiện sử dụng dns từ mạng external đến một mạng local (dns_external_resolvers) ; hoặc phát hiện máy trong mạng local dùng DNS external (dns_open_resolvers)
* Pattern finder: so sánh tìm kiếm mẫu flow data cho trước, với ví dụ là mẫu flow data của SSH authen attack dạng Brute Force
* Ports Scan detection
* SSH Authentication simple: phát hiện tấn công từ điển ssh authentication

## <a name="Các thành phần trong soucre code"></a> 3. Các thành phần trong soucre code

Source code: https://github.com/CSIRT-MU/Stream4Flow

Bao gồm 3 thư mục: applications, provisioning, web-interface; 2 file giới thiệu README.md LICENSE

### Thư mục provisioning:
Chứa các hướng dẫn triển khai các thành phần hệ thống, bao gồm deploy cluster từ ansible hoặc all-in-one dùng vagrant (build các máy ảo dùng virtual-box ảo hóa)

Có 3 thành phần hợp thành hệ thống bao gồm: 

* Producer: cài đặt ipfixcol, kafka.

* Cụm spark: cài một spark master và các slave. Cũng cài các thư viện dùng kết nối với kafka và các application cho statistics, detect

* Consumer: cài elk-stack (Elasticsearch, Logstash , Kibana). Đồng thời nó cũng có có tạo giao diện Web riêng Stream4Flow (port :80) hiển thị kết quả các application

### Thư mục applications:

Chứa các chương trình cho statistics, detection đã kể trên, viết từ pySpark. Thư mục này khi cài đặt sẽ được copy vào cụm spark. Đồng thời nó cũng hỗ trợ template để ta có thể tự viết các applications riêng.

### Thư mục web-interface: 

Chứa Stream4Flow web interface, thư mục này khi cài đặt sẽ được copy vào Consumer. 

Sau khi cài xong consumer, đầu tiên ta chỉ thấy được web insterface cho ứng dụng "Protocol statistics", các ứng dụng khác muốn hiện giao diện cần cấu hình thêm. 

 
 (Các ứng dụng DDOS detection hoàn toàn chưa làm giao diện, em chạy ứng dụng xong mà không biết xem kết quả kiểu gì)
 
## <a name="Cách thức hoạt động"></a> 4. Cách thức hoạt động

Bước 1: Tại Producer: 
Kafka sau khi cài xong sẽ tạo sẵn 2 topic (topic là khái niệm giống một hàng đợi thông điệp trong message-queue). Một tên ipfix.entry-nhận input. Một tên results.output-nhận output. 

Ipfixcol nhận netflow/ipfix data từ udp cổng 4739 , đọc và đưa kết quả ra dưới dạng json, gửi đến ngay topic ipfix.entry trong Kafka.

![](./images/stream4flow-producer.PNG)

Ở đây hệ thống đã tự cấu hình, ta không cần làm gì thêm. Tuy nhiên để verify lại , ta gửi ipfix package đến cho producer theo lệnh:

```
ipfixsend -i test-data.ipfix -d 192.168.2.54 -p 4739 -t UDP -n 10
```
Trong đó test-data.ipfix đã chuẩn bị trước, có thể lấy tại https://github.com/CSIRT-MU/Stream4Flow/blob/master/provisioning/test/integration/roles/integration-test/files/test-data.ipfix

Kiểm tra kafka topic ipfix.entry ta sẽ thấy kết quả xử lý của Ipfixcol đã bắn vào trong kafka.

```
/opt/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic ipfix.entry --from-beginning

{"@type": "ipfix.entry", "ipfix.octetDeltaCount": 59, "ipfix.packetDeltaCount": 1, "ipfix.protocolIdentifier": 17, "ipfix.ipClassOfService": 0,
"ipfix.sourceTransportPort": 8278, "ipfix.sourceIPv4Address": "3.0.0.0", "ipfix.ingressInterface": 0, "ipfix.destinationTransportPort": 53, "ipfix.destinationIPv4Address": "8.8.8.8", "ipfix.egressInterface": 0, "ipfix.samplingInterval": 0,"ipfix.samplingAlgorithm": 0, "ipfix.ipVersion": 4, "ipfix.flowStartMilliseconds": 1489141087955, "ipfix.flowEndMilliseconds": 1489141087955}
{"@type":  "ipfix.entry","ipfix.octetDeltaCount": 20,...
```

Các thông tin định dạng json ở trên cũng là dữ liệu mà spark phải xử lý với.

Bước 2: Tại Spark master.
các application của Stream4Flow framework đã được copy sang sparkMaster tại đường đẫn /home/spark/applications. Tại spark, ta chỉ định chạy một (hoặc nhiều) application. Vd:

```
./run-application.sh ./statistics/protocols_statistics/spark/protocols_statistics.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output
```
Lệnh trên để chạy ứng dụng protocols_statistics. Phân tích câu lệnh trên ta có:
* -iz: input_zookeeper   địa chỉ của zookeeper đang quản lý kafka trên producer
* -it: input_topic       kafka topic đầu vào . ipfix.entry
* -oz: output_zookeeper  địa chỉ kafka-broker nơi đẩy dữ liệu đã xử lý đến.
* -ot: output-topic      kafka topic đầu ra. results.output

Sau khi chạy ứng dụng, spark thực hiện đọc message trên topic ipfix.entry, xử lý nó và đưa kết quả ra topic results.output.

![](./images/stream4flow-spark.PNG)

Trường hợp có cùng nhiều application được chạy, các bản ghi output ra trên topic results.output phân biệt cho mỗi ứng dụng bởi trường "@type". 

Ví dụ ta cùng chạy 2 ứng dụng đọc kafka topic results.output thì được:

```
/opt/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic results.output --from-beginning

{"@type": "host_stats","src_ip":"3.0.0.0","stats":{"total":{"packets":40,"bytes":5600,"flow":40},"avg_flow_duration":0.0,"dport_count":2,"peer_number":2}}

{"@type": "protocols_statistics", "protocol": "udp", "flows": 1, "packets": 2, "bytes": 248}

{"@type": "protocols_statistics", "protocol": "other", "flows": 20, "packets": 20, "bytes": 1680}

{"@type": "protocols_statistics", "protocol": "tcp", "flows": 60, "packets": 60, "bytes": 360480}
```

Bước 3: Tại consummer: 

![](./images/stream4flow-consumer.PNG)

Logstash liên tục thu thập dữ liệu từ kafka của producer nạp vào search-engine elasticsearch, ta có thể thấy điều này qua config của logstash /etc/logstash/conf.d/kafka-to-elastic.conf
```
input {
	kafka{
	    topics => ["results.output"]
        bootstrap_servers => "producer:9092"
        codec => "json_lines"
    }
}
output{
	elasticsearch{
                hosts => "localhost:9200"
                codec => json
                index => "spark-%{+YYYY.MM.dd}"
		template => "/etc/logstash/conf.d/templates/spark-elasticsearch-template.json"
		template_name => "spark"
        }
	stdout{
		codec => rubydebug
	}
}
```

Sau khi dữ liệu vào elasticsearch sẽ được query tới bởi Steam4Flow web interface. Đổ ra view cho người dùng xem kết quả của application.

Nếu thành thạo về elaticsearch ta có thể vào consummer Kibana web interface (port :5601) để tự viết query lấy (em thì không biết dùng elaticsearch tí gì).

Drashboard của stream4flow:
![](./images/stream4flow-drashboard.PNG)

## <a name="Mô hình cài đặt- kết hợp sahara"></a> 5. Mô hình cài đặt- kết hợp sahara

Với mô hình cài đặt như dưới đây:

![](./images/sahara_stream4flow1.PNG)

Trong đó:

Producer và consumer được cài cùng trên một host 2.54, dùng ansible của stream4flow để deploy

Cụm spark được deploy bằng sahara, sau khi sahara chạy xong, để chạy ứng dụng của stream4flow, ta cần thực hiện trên spark master và slave:
* Cài đặt scala

```
sudo apt-get install scala zip
```

* Cài đặt pip
```
sudo apt-get install python-pip python-dev
```

* Cài dowload các application của steam4flow về:

```
sudo apt-get install -y git
git clone https://github.com/CSIRT-MU/Stream4Flow.git
cp -r Stream4Flow/applications .
```

* Thêm module kết nối kafka và spark
```
wget -O spark-streaming-kafka-assembly.jar http://search.maven.org/remotecontent?filepath=org/apache/spark/spark-streaming-kafka-0-8-assembly_2.11/2.1.1/spark-streaming-kafka-0-8-assembly_2.11-2.1.1.jar
cp spark-streaming-kafka-assembly.jar /opt/spark/jars/
```

* Cài đặt các python package đi cùng application

```
sudo -i
pip install ujson termcolor ipaddress netaddr kafka
```

* Tạo file script chạy cho các stream4flow application:

```
cd /home/ubuntu/applications
nano run-application.sh

#!/bin/bash

if [ -z "$1" ] 
then
    echo "You must specify the application with arguments"
    exit 1;
fi
APPLICATION=$@
MODULESDIR=$(dirname "$1")/modules

# Output colors
GREEN='\033[0;32m'
NC='\033[0m' # No Color
PYFILES=''

if [ -d $MODULESDIR ]
then
    echo -e "${GREEN}[info] Creating zip with all modules${NC}"
    zip -r $MODULESDIR{.zip,}
    PYFILES="--py-files $MODULESDIR.zip"
fi

echo -e "${GREEN}[info] Running application $APPLICATION...${NC}"

/opt/spark/bin/spark-submit --total-executor-cores 2 --executor-memory 1G $PYFILES $APPLICATION


chmod +x run-application.sh
```

* Sửa config: tại file /opt/spark/conf/spark-defaults.conf thêm dòng:

```
spark.jars      /opt/spark/jars/spark-streaming-kafka-assembly.jar
```

## <a name="Chạy các application"></a> 6. Chạy các application

Các lệnh đều được thực hiên trên spark master, user ubuntu

### Statistics

* Protocol statistics

```
cd /home/ubuntu/applications/
./run-application.sh ./statistics/protocols_statistics/spark/protocols_statistics.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output
```

output sẽ đưa ra:

```
{"src_ip":"192.168.2.54","stats":{"total":{"packets":4,"bytes":240,"flow":1},"avg_flow_duration":5.009,"dport_count":1,"peer_number":1},"@type":"host_stats"}
```

* Host statistics:

```
cd /home/ubuntu/applications/
./run-application.sh ./statistics/hosts_statistics/spark/host_stats.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -ln "0.0.0.0/0"
```

* DNS statistics

```
cd /home/ubuntu/applications/
./run-application.sh ./statistics/dns_statistics/spark/dns_statistics.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -ln "0.0.0.0/0"
```

### Detection

* DDOS detection

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/ddos/spark/detection_ddos.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -nf "192\.168\..+"
```

output đưa ra:

```
{"attackers":["192.168.2.192"],"dst_ip":"192.168.2.54","@type":"detection.ddos","longratio":1.5967693688,"shortratio":1.5996403956}
```

* DNS Servers Usage: 

(dns_external_resolvers) 

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/dns_external_resolvers/spark/dns_external_resolvers.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -ln 192.168.0.0/16
```

output: không có, lý do: spark sẽ có lọc ra các luồng vào có element ipfix.DNSRData tuy nhiên element này lại không hề xuất hiện trong input vào.
(dns_open_resolvers)

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/dns_open_resolvers/spark/dns_open_resolvers.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -ln 192.168.0.0/16
```

* Pattern finder: 

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/pattern_finder/spark/pattern_finder.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output -c ./detection/pattern_finder/spark/configuration.yml
```

* Ports Scan detection

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/ports_scan/spark/ports_scan.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output
```

* SSH Authentication simple

```
cd /home/ubuntu/applications/
./run-application.sh ./detection/ssh_auth_simple/spark/ssh_auth_simple.py -iz producer:2181 -it ipfix.entry -oz producer:9092 -ot results.output
```





