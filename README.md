Golang:

> 反射

```go
var subscribeMap = map[string]interface{}{
		"HisOutpatientPrescription": models.HisOutpatientPrescription,
		"HisInHospitalPrescription": models.HisInHospitalPrescription,
		"HisHomeBedPrescription":    models.HisHomeBedPrescription,
}

for k, v := range subscribeMap {
    
    m := map[string]interface{}{k: v}
    _, _ := Call(m, k, params)
    
}

func Call(m map[string]interface{}, name string, params ...interface{}) ([]reflect.Value, error) {
    f := reflect.ValueOf(m[name])
    if len(params) != f.Type().NumIn() {
        return nil, errors.New("the number of input params not match!")
    }
    in := make([]reflect.Value, len(params))
    for k, v := range params {
        in[k] = reflect.ValueOf(v)
    }
    return f.Call(in), nil
}
```

> 穿戴设备

```go
package main

import (
    "flag"
    "fmt"
    "net"
    "os"
    "strings"
)

var host = flag.String("host", "", "host")
var port = flag.String("port", "7890", "port")
func main() {
    // 解析参数
    flag.Parse()
    var l net.Listener
    var err error // 监听
    l, err = net.Listen("tcp", *host+":"+*port)
    if err != nil {
        fmt.Println("Error listening:", err)
        os.Exit(1)
    }
    defer l.Close()
    fmt.Println("Listening on " + *host + ":" + *port)
    for {
        // 接收一个client
        conn, err := l.Accept()
        if err != nil {
            fmt.Println("Error accepting: ", err)
            os.Exit(1)
        }
        fmt.Printf("Received message %s -> %s \n", conn.RemoteAddr(), conn.LocalAddr())
        // 执行
        go handleRequest(conn)
    }
}
// 处理接收到的connection
func handleRequest(conn net.Conn) {
    ipStr := conn.RemoteAddr().String()
    fmt.Println(ipStr)
    for {
        var buf [128]byte
        n, _ := conn.Read(buf[:])
        str := string(buf[:n])
        fmt.Printf("Received message from client,data:%v\n", str)
        //给表应答数据
        d := strings.Replace(str, "T", "S", -1)
        b := []byte(d)
        conn.Write(b)
    }
}
```



Js:

> DICOM web - app.js

```javascript
var express = require('express');
var app = express();
var bodyParser = require('body-parser');
var dicom = require('./dicom.js');
//引用bodyParser 这个不要忘了写
app.use(bodyParser.json());
// for parsing application/json
app.use(bodyParser.urlencoded({
    extended: true
}));
// for parsing application/x-www-form-urlencoded
//设置跨域访问
app.all('*',
function(req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "X-Requested-With");
    res.header("Access-Control-Allow-Methods", "PUT,POST,GET,DELETE,OPTIONS");
    res.header("X-Powered-By", ' 3.2.1');
    res.header("Content-Type", "application/json;charset=utf-8");
    next();
});
app.get('/open', 
async function(req, res) {
    
    let accessionNumber = req.query.AccessionNumber;
    let body = await dicom.checkExistInlocal(accessionNumber);
    if (body.length == 0){//本地服务器没有找到
        console.log('没有找到在本地资源库');
        let message = await retrievePatientByOne(accessionNumber);
        console.log(message);
        let message2 = await dicom.checkExistInlocal(accessionNumber);
        res.redirect('http://ip:port/osimis-viewer/app/index.html?study='+message2[0]);
        res.end(); 
    }else{
       res.redirect('http://ip:port/osimis-viewer/app/index.html?study='+body[0]);
       res.end(); 
    }
});

app.post('/wdltest',
function(req, res) {
    console.log(req.stack);
    console.log(req.body);
    console.log(req.url);
    console.log(req.query);
    res.json(req.body)
})
//配置服务端口
var server = app.listen(3001,
function() {
    var host = server.address().address;
    var port = server.address().port;
    console.log('Example app listening at http://%s:%s', host, port);
})


let retrievePatientByOne = async function(accessionNumber) {
    //获取查询key
    let queryString = await dicom.getQueryString(accessionNumber);
    //根据查询key和id进行这一次检查的retrieve
    let body = await dicom.performingRetrieveByOne(queryString);

    return body;
}
```

> DICOM - dicom.js

```javascript
var dicom = {
    //得到查询Key
    getQueryString: function(accessionNumber) {
        const request = require('request');
        let url = "http://ip:port/modalities/jinshan/query/";
        let patient = {
            AccessionNumber: accessionNumber
        };
        let requestData = {
            Level: "Study",
            Query: patient
        };
        return new Promise(function(resolve, reject) {
            request({
                url: url,
                method: "POST",
                json: true,
                headers: {
                    "content-type": "application/json",
                },
                body: requestData
            },
            function(error, response, body) {
                if (!error && response.statusCode == 200) {
                    resolve(body.ID);
                } else {
                    reject(error);
                }
            });
        });
    },
    //根据某一次检查进行Retrieve
    performingRetrieveByOne: function(queryString) {

        const request = require('request');
        let url = "http://ip:port/queries/" + queryString + "/answers/0"+"/retrieve";
        console.log(url);
        let requestData = {
            TargetAet: "bsoft3",
            Synchronous: true
        };
        return new Promise(function(resolve, reject) {
            request({
                url: url,
                method: "POST",
                json: true,
                //body: JSON.stringify(requestData)
                headers: {
                    "content-type": "application/json",
                },
                body: requestData
            },
            function(error, response, body) {
                if (!error && response.statusCode == 200) {
                    resolve(body);
                } else {
                    reject(error);
                }
            });
        });
    },
    //检索本地服务器是否有改检查
    checkExistInlocal: function(accessionNumber){
        const request = require('request');
        let url = "http://ip:port/tools/find";
        let requestData = {
            Level: "Study",
            Query: {
                AccessionNumber: accessionNumber
            }
        };
        return new Promise(function(resolve, reject) {
            request({
                url: url,
                method: "POST",
                json: true,
                headers: {
                    "content-type": "application/json",
                },
                body: requestData
            },
            function(error, response, body) {
                if (!error && response.statusCode == 200) {
                    resolve(body);
                } else {
                    reject(error);
                }
            });
        });
    }
}
module.exports = dicom;
```

Python:

> 经纬度

```python
file_loc = "D:\data.xlsx"
df = pd.read_excel(file_loc, usecols=[0, 1 ,3 ,57])
# 创建一个空的dataframe
dr = pd.DataFrame(columns=["ORG_CODE","FPRN","FNAME","DZ", "LNG", "LAT","CITYCODE"])  
for index, row in df.iterrows():
    base = "http://api.map.baidu.com/geocoder/v2/?address=" + row[
        "DZ"] + "&output=json&ak=...."
    response = requests.get(base)
    status = response.json()['status']
    if status == 0:
        result = response.json()['result']
        lng = result['location']['lng']  # 经度
        lat = result['location']['lat']  # 纬度
        cityUrl = "http://api.map.baidu.com/geocoder/v2/?ak=..."+"&location=" + str(lat) + ","+ str(lng) + "&output=json&pois=1"
        responseUrl = requests.get(cityUrl)
        resultUrl = responseUrl.json()['result']
        cityCode = resultUrl['cityCode'] # 城市
        print(index,row["FPRN"],row["DZ"],lng,lat,cityCode)
        dr = dr.append({'ORG_CODE': row["ORG_CODE"], 'FPRN': row["FPRN"],'FNAME': row["FNAME"],'DZ': row["DZ"], 'LNG': lng, 'LAT': lat ,'CITYCODE': cityCode}, ignore_index=True)
    else:
        dr = dr.append({'ORG_CODE': row["ORG_CODE"], 'FPRN':"",'FNAME':"",'DZ': row["DZ"], 'LNG': "", 'LAT': "",'CITYCODE':""}, ignore_index=True)
print(dr)
dr.to_csv("D:\result.csv", encoding="gbk", index=False)  # 写入到csv时，不要将索引写入index = False
```

Java:

> HL7

```java
package ca.uhn.hl7v2.examples;

import ca.uhn.hl7v2.DefaultHapiContext;
import ca.uhn.hl7v2.HL7Exception;
import ca.uhn.hl7v2.HapiContext;
import ca.uhn.hl7v2.model.Message;
import ca.uhn.hl7v2.model.v22.message.ADT_A01;
import ca.uhn.hl7v2.model.v26.message.ORU_R01;
import ca.uhn.hl7v2.model.v26.segment.MSH;
import ca.uhn.hl7v2.parser.EncodingNotSupportedException;
import ca.uhn.hl7v2.parser.Parser;
import ca.uhn.hl7v2.parser.PipeParser;
import ca.uhn.hl7v2.util.Terser;

/**
 * Example code for parsing messages.
 * 
 * @author <a href="mailto:jamesagnew@sourceforge.net">James Agnew</a>
 * @version $Revision: 1.1 $ updated on $Date: 2007-02-19 02:24:46 $ by $Author: jamesagnew $
 */
public class ExampleParseMessages
{

    /**
     * A simple example of parsing a message
     */
    public static void main(String[] args) {
        String msg = "MSH|^~\\&|MINDRAY_VS900^00A0370098007A80^EUI-64||||20191206113452||ORU^R01^ORU_R01|1914|P|2.6|||AL|NE||UNICODE UTF-8|||IHE_PCD_001^IHE PCD^1.3.6.1.4.1.19376.1.6.1.1.1^ISO\r"
                + "PID|||^^^Hospital^PI||^^^^^^L\r"
                + "PV1||I||||||||||||||||||||||||||||||||||||||||||20191201\r"
                + "OBR|1|1914^MINDRAY_VS900^00A0370098007A80^EUI-64|1914^MINDRAY_VS900^00A0370098007A80^EUI-64|182777000^monitoring of patient^SCT|||20191206113452\r"
                + "OBX|1|NM|150456^MDC_PULS_OXIM_SAT_O2^MDC|1.3.1.150456|99|262688^MDC_DIM_PERCENT^MDC|||||R|||20191206113452||||00A0370098007A80^^00A0370098007A80^EUI-64\r"
                + "OBX|2|NM|149530^MDC_PULS_OXIM_PULS_RATE^MDC|1.3.1.149530|94|264864^MDC_DIM_BEAT_PER_MIN^MDC|||||R|||20191206113452\r"
                + "OBX|3|NM|150488^MDC_BLD_PERF_INDEX^MDC|1.3.1.150488|0.94|262688^MDC_DIM_PERCENT^MDC|||||R|||20191206113452\r"
                + "OBX|4|NM|150301^MDC_PRESS_CUFF_SYS^MDC|1.1.9.150301|147|266016^MDC_DIM_MMHG^MDC|||||R|||20191206111016|||^APERIODIC\r"
                + "OBX|5|NM|150303^MDC_PRESS_CUFF_MEAN^MDC|1.1.9.150303|109|266016^MDC_DIM_MMHG^MDC|||||R|||20191206111016|||^APERIODIC\r"
                + "OBX|6|NM|150302^MDC_PRESS_CUFF_DIA^MDC|1.1.9.150302|90|266016^MDC_DIM_MMHG^MDC|||||R|||20191206111016|||^APERIODIC\r"
                + "OBX|7|NM|188496^MDC_TEMP_AXIL^MDC|1.2.5.188496|35.1|268192^MDC_DIM_DEGC^MDC|||||R|||20191206113415|||^APERIODIC";
        /*
         * The HapiContext holds all configuration and provides factory methods for obtaining
         * all sorts of HAPI objects, e.g. parsers. 
         */
        HapiContext context = new DefaultHapiContext();
        
        /*
         * A Parser is used to convert between string representations of messages and instances of
         * HAPI's "Message" object. In this case, we are using a "GenericParser", which is able to
         * handle both XML and ER7 (pipe & hat) encodings.
         */
        Parser p = context.getGenericParser();

        Message hapiMsg;
        try {
            // The parse method performs the actual parsing
            hapiMsg = p.parse(msg);
        } catch (EncodingNotSupportedException e) {
            e.printStackTrace();
            return;
        } catch (HL7Exception e) {
            e.printStackTrace();
            return;
        }
        ORU_R01 adtMsg = (ORU_R01)hapiMsg;
        //获取OBX的数量
        int obxNum = adtMsg.getPATIENT_RESULT().getORDER_OBSERVATION().getOBSERVATIONReps();
        //循环OBX每一项
        for (int i =0;i<obxNum; i++){
            //key
            System.out.println(adtMsg.getPATIENT_RESULT().getORDER_OBSERVATION().getOBSERVATION(i).getOBX().getObservationIdentifier().getCwe2_Text());
            //value
            System.out.println(adtMsg.getPATIENT_RESULT().getORDER_OBSERVATION().getOBSERVATION(i).getOBX().getObservationValue()[0].getData());
        }

    }



}
```

> yaml

```java
public class ConfMeta {
    private ArrayList<Hospital> hospitals;
   //Getting and Setting
}
public class Device {
    private String devid;
    private String ip;
    private String port;
    private String devname;
    //Getting and Setting
}
public class Hospital {
    private String orgid;
    private String orgname;
    private ArrayList<Device> devices;
    //Getting and Setting
}

public  static ConfMeta getConf(){
FileInputStream fileInputStream = null;
ConfMeta confMeta =null;
try {
       //实例化解析器
       Yaml yaml = new Yaml();
       //配置文件地址
       File file = new File(System.getProperty("user.dir")+File.separator+"conf"+File.separator+"device.yml");
       System.out.printf(System.getProperty("user.dir"));
       fileInputStream = new FileInputStream(file);
      //装载的对象，这里使用Map, 当然也可使用自己写的对象
       confMeta = yaml.loadAs(fileInputStream, ConfMeta.class);
       System.out.printf(confMeta.toString());
    } catch (FileNotFoundException e) {
            e.printStackTrace();
       }
      return confMeta;
}
```

```yaml
hospitals:
 -
  orgid: 1
  orgname: ...
  devices:
     -
        devid: 1001
        ip: 10.1.1.1
        port: 8080
        devname: 监控1
     -
        devid: 1002
        ip: 10.1.1.2
        port: 8080
        devname: 监控2

 -
  orgid: 2
  orgname: ...
  devices:
     -
       devid: 2001
       ip: 10.2.2.1
       port: 8080
       devname: 监控1
     -
       devid: 2002
       ip: 10.2.2.2
       port: 8080
       devname: 监控2
     -
       devid: 2003
       ip: 10.2.2.3
       port: 8080
       devname: 监控3
```

