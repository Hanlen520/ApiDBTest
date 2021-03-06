# Data driving for interface testing[接口测试之数据驱动]
一、简介：

1、使用java发起http/https请求测试接口，快速稳定

2、应用mybatis框架，将测试用例放在数据库维护，方便小组成员共同参与

3、同时生成html和Excel测试报告，报告内容详细清晰，错误日志一目了然

4、方便直接部署在jenkins平台自动测试，一键执行测试用例，还可定时执行

5、支持随时更新/增加测试用例，无需修改代码，可定向设置用例的有效性

6、直接在数据库维护测试套件，可定向测试单个接口或者单条用例，使用简单

二、编写接口参数类：

    ①每个接口编写一个类，写好所有参数类型
    
    ②重写toString（）方法，使其返回结果为请求参数格式的数据
    
    ③添加一个Result（）方法，使其返回结果为ExcelReport数组结果
    
    public class postcode {
    public int ID;
    public String CaseName;
    public String RequestMethod;
    public String postcode;
    public String areaname;
    public String appkey;
    public String sign;
    public String format;
    public String ExpectResult;
    public String Effective;

    @Override
    public String toString(){
        if(!postcode.equals("null")){
            return "?app=life.postcode&postcode="+postcode+"&appkey="+appkey+"&sign="+sign+"&format="+format;
        }else {
            return "?app=life.postcode&areaname="+areaname+"&appkey="+appkey+"&sign="+sign+"&format="+format;
        }

    }

    //组装报告的结果数组(此方法由于使用频率极高，故建议写成公共方法)
    public String[] Result(){
        String[] result =new String[8];
        result[0]= ConfigFile.getUrl("test.url");
        result[1]="PostCode";
        result[2]=CaseName;
        result[3]=RequestMethod;
        result[4]=this.toString();
        result[5]=ExpectResult;
        return result;
    }
    }

三、读取测试用例：

    在src\main\resources\mapper\SQLMapper.xml中按接口用例表添加相应的读取用例的sql
    
 示例：
 
    #查询接口用例总数：
     <select id="postcodecount"  resultType="Integer">
         select count(*) from postcode;
    </select>
    
    #查询接口单条用例：
    <select id="postcode" 【接口名称即表名】 parameterType="Integer" 【请求参数类型，即读取第几条用例】 resultType="CaseData.postcode" 【返回      的参数类型】>
        select * from postcode where ID=#{id} 【使用#{   }的形式参数化变量】;
    </select>

四、数据库用例设计：

每个接口设计一张表，结构统一如下：

    ①表名==接口名称
    
    ②ID==用例编号
    
    ③CaseName==用例名称
    
    ④RequestMethod==请求方法（post/get）
    
    ⑤data==接口参数字段(一个参数一个字段)
    
    ⑥ExpectResult==预期结果
    
    ⑦Effective==用例是否有效

数据库用例表结构示例：
    
    登录接口参数{用户名|密码|用户类型}
    [USER]
    | 用例编号 |   用例名称 |      请求方式    |   用户名  |   密码    | 用户类型 |    预期结果   |  用例是否有效 | 
    |   ID    |  CaseName  |  RequestMethod  | uasername | password |  type   |  ExpectResult |   Effective  |
    
        1     正确账号密码登录      post         zhangsan    123456       1     equals[success,1]       T


五、新增接口后编写CASE:
    
    在BaseCase目录下的CaseUtil文件中修改getcasedata方法即可：
    
    创建一个接口类对象勇于接受测试用例数据，然后给BaseCase对象赋值执行用例
    
        BaseCase casedata=new BaseCase();
        if (casename.equals("postcode")){
            postcode casevalue= DatabaseUtil.getSqlSession().selectOne("postcode",i);
            casedata.setRequest(casevalue.toString()); //请求参数
            casedata.setType(casevalue.getRequestMethod().trim().toLowerCase()); //读取请求类型
            casedata.setExcepct(casevalue.getExpectResult().trim()); //读取预期结果
            casedata.setIsdo(casevalue.getEffective().trim());//是否执行用例
            casedata.setResultExcel(casevalue.Result());//Excel报告结果
        }


    其他关键类说明：

    1.InitTest用于生成报告，StartTest类需继承它：
    
    public class InitTest {

    @BeforeTest
    public void beforetest(){
        InitExcelReport.InitExcel();
    }

    @AfterTest
    public void aftertest(){
        new InitHtmlReport().CreatHtmlReport();
    }
    }
    
    2.StartTest用于执行用例,调用BaseCase基类处理数据、执行用例、记录结果：
    
    public class StartTest extends InitTest{

    private static Logger logger = LoggerFactory.getLogger(StartTest.class);

    @Test(dataProvider="TestData",dataProviderClass= DataSource.class)
    public  void StartTest(String caseqty ,String casename){
        BaseCase baseCase=new BaseCase();
        try {
        【先读取用例总数，然后for（）循环执行用例】
            baseCase.executecase(caseqty,casename);
        } catch (IOException e) {
            logger.error("\n本次测试执行过程中出现异常，具体原因如下：\n\n"+ ExceptionUtil.getStackTrace(e));
        }
    }
    }
    
    3.DataSource使用数据驱动读取需要测试套件内容，驱动测试执行：
        
    
六、结果断言：
    
    预期结果的格式：
    比较方式[字段名：预期结果];比较方式[字段名：预期结果] 
    例： contain[key：vlaue];equals[key：vlaue]
    
    1.可自定义支持结果长度的比较，需要根据实际情况在AnylistJson类修改getsize（）方法即可
    2.同时支持多个字段的校验，使用 ; 分隔即可
    3.断言字典如下：
        equals  ---  等于value
        contain ---  包含value
        length  ---  长度等于（未具体实现）
        start   ---  以value开头
        end     ---  以value结尾       

七、使用testng数据驱动从数据库直接读取测试套件，避免反复修改testng.xml：
    
    1.新建测试套件表suitcase：
       public int id;
        public String casename;//查询接口名称的sql的id
        public String caseqty;//查询接口对应的用例数量的sql的id
        public String effictive;//是否执行该接口用例

         @Override
        public String toString(){
        return "编号:"+id+",测试接口名称:"+casename+",接口用例数量:"+caseqty+",本次是否执行(T-执行/F-不执行):"+effictive;
        }
        
        
     2.在/TestCase/DataSource下使用数据驱动读取测试套件表，返回规定的object[][]格式参数，驱动测试执行
     
     3.新建testurl表（一个字段即可），指定当前的测试所需要读取的测试环境：
        测试环境配置在/resource/application.properties中配置
        SIT--测试环境
        UAT--验收环境
        PRO--生产环境
     
     4.通过StartTest方法一键执行所有用例
        

八、ExcelReport数据结构：

    ①ID==用例编号（从1自增）
    
    ②CaseName==用例名称（数据库读取）
    
    ③ApiName==接口名称（表名）
    
    ④TestURL==测试地址（代码里配置）
    
    ⑤RequestData==请求参数（数据库读取）
    
    ⑥ExpectResult==预期结果（数据库读取）
    
    ⑦ResponseData==响应结果
    
    ⑧Result==测试结果（Pass/Fail，默认为Fail）
    
    ⑨每条用例执行结束后组装ExccelReport和HtnlReport数据，并写入结果
    
九、部署jenkins持续集成环境,执行用例：
    
    jenkins部署自行百度，部署好新建基于maven的任务，执行指定的testng.xml文件即可：

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd">
    <suite name="API自动化测试" >
    <test name="测试">
        <classes>
            <class name="TestCase.StartTest"></class>
        </classes>
    </test>
    </suite>
    
    执行结束后在TestReport查看Excel报告和TestReport查看html报告
    
    
 十、部署测试报告小平台：
 
    在jenkins服务器上搭建一个tomcat环境，然后在jenkins上通过shell脚本将每次执行的报告拷贝至/webapps/ROOT目录下，则
    可通过链接查看测试报告：
    
    shell命令参考（百度可查）：
    
    result=$(curl -s http://ip:端口/job/项目名称/lastBuild/buildNumber  --user 用户名：密码)   --result 即jenkins任务执行编号，据
    此创建文件夹

    mkdir  在tomcat目录的webapps/ROOT下创建/$result文件夹

    cp /root/.jenkins/workspace/项目名称/TestReport/index.html   ./webapps/ROOT/$result/index.html
    
    最后使用链接即可访问测试报告：
    http://IP:端口/result/index.html
    
    
    
十一、执行结束后可使用插件自动发送邮件：


    1.jenkins发送邮件可自行百度
    
    2.亦可新建任务自己通过脚本实现邮件发送
    
