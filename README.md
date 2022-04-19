# OGS
A game server side framework
# Preface


## Description of test program

The test program is based on my OGS components. The test program is based on my OGS components. Now, I will give the steps of implementations.

# Steps

## 1 Create a console program based on dotnet core with VS, as shown in the following figure:

![image-20220223180752496](C:\Users\majp\AppData\Roaming\Typora\typora-user-images\image-20220223180752496.png)

## 2 Write the ITestorleans interface, but this interface will not contain any methods, just add a new type label.

> Notice
>
> ITestorleans  is just an empty interface. I use it only as a different type declaration.

```c#
using OGSOrleans.inf;
using System;
using System.Collections.Generic;
using System.Text;

namespace TestOrleans
{
    public interface ITestOrleans : IGeneralGrainCaller
    {
    }
}
```

## 3 Write core class,Testgrain 

## Which is used to implement distributed services. Through this service, the client can access any method of the object marked with GrainClassMethod annotation in the program where the service is located.

> Notice
>
> A server can have only one empty service interface implementation class. Other service functions completely depend on the annotation: **GrainClassMethod**.

```c#
using OGSCommon.attributes;
using OGSOrleans.grains;
using OGSOrleans.streams;
using Orleans.Streams;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;

namespace TestOrleans
{
    public class TestGrain : LogicGrainUnit, ITestOrleans, IOrleansStreamProvider<string>
    {
        private int ticks = 0;
        private StreamProducer<string> producer;
        private bool trigProducer = false;
        private int triggerTimes = 0;

        public IAsyncStream<string> GetStreamEx(string id, string ns, string provider)
        {
            return GetStream<string>(id, ns, provider);
        }

        public override async Task OnStart()
        {
            this.TimerStarted = true;
            producer = new StreamProducer<string>(this);
            await producer.Start("8abf0545-5944-476a-913d-8719b6afde4f", "namespaceX");
        }

        public override async Task OnStop()
        {
            await producer.Stop();
        }

        public override Task OnUpdate()
        {
            if (trigProducer)
            {
                if (++ticks > 100)
                {
                    ticks = 0;
                    producer.Fire(DateTime.Now.ToString());
                    Console.WriteLine($"msg produced, left {triggerTimes}");
                    if (--triggerTimes < 1)
                        trigProducer = false;
                }
            }
            return Task.CompletedTask;
        }
        [GrainClassMethod]
        public void StartProduce(int n)
        {
            triggerTimes = n < 1 ? 1 : n;
            ticks = 0;
            trigProducer = true;
            Console.WriteLine($"StartProduce times {triggerTimes} , trig {trigProducer}");
        }

        [GrainClassMethod]
        public void TestFunc(string name, int value)
        {
            Console.WriteLine($"TestFunc {name} , {value}");
        }
        [GrainClassMethod]
        public int AnswerFunc(string name, int value)
        {
            Console.WriteLine($"TestFunc {name} , {value}");
            return value + 100;
        }
    }
}

```

## 4 Write another ordinary class. 

This class contains methods annotated with GrainClassMethod, which can be called directly by the client.

>  **Notice**
>
>  **This class does not inherit from the grain class, but is a normal class. This is the role of my framework, which creates any number of service functions based on annotations, independent of interface programming.**

```c#
using System;
using System.Collections.Generic;
using System.Text;
using OGSCommon.attributes;

namespace TestOrleans
{
    internal class AnotherPlainClassBeCalledWithGrainInterface
    {
        [GrainClassMethod]
        public void TestFunc(string name, int value)
        {
            Console.WriteLine($"TestFunc(AnotherPlainClassBeCalledWithGrainInterface) {name} , {value}");
        }
    }
}

```

## 5 Writing client code

```c#
using OGSOrleans.shell;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TestOrleans
{
    class Program
    {
        static TestGrainClient tgc;
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
            if (args.Length < 1)
                return;

            switch(args[0])
            {
                case "tcsm":
                    TestConsume();
                    break;
                case "tpd":
                    TestProducer();
                    break;
                case "gsvr":
                    TestGrainSvr();
                    break;
                case "gcli":
                    TestGrainCli();
                    break;
            }

            Console.WriteLine("press any key to exit.");
            Console.ReadLine();
        }
        static void TestConsume()
        {
            try
            {
                GrainsClient.Initial();
                tgc = new TestGrainClient();
                ITestOrleans test = GrainsClient.Inst().GetGrainT<ITestOrleans>("testid1");
                test.Call("TestGrain.TestFunc", "Call Test", 100).Wait();
                Console.WriteLine("Invoke : " + test.Invoke("TestGrain.AnswerFunc", "Invoke Test", 200).Result);
                //Task.Run(async () => await tgc.Start()).Wait();
            }
            catch(Exception e)
            {
                Console.WriteLine(e.ToString());
            }
        }
        static void TestProducer()
        {
            GrainsServer.RunServer();
        }

        static void TestGrainSvr()
        {
            GrainsServer.RunServer();
        }
        static void TestGrainCli()
        {
            GrainsClient.Initial();
            //tgc = new TestGrainClient();
            //Task.Run(async () => await tgc.Start()).Wait();

            int[] ids = new int[100];
            for(int i = 0; i < 100; ++i )
            {
                ids[i] = i + 1;
            }

            var rand = new Random();

            for (int i = 0; i < 10000; ++i)
            {
                try
                {
                    ITestOrleans test = GrainsClient.Inst().GetGrainT<ITestOrleans>($"testGainId{ids[rand.Next(0, 100)]}");
                    test.Call("AnotherPlainClassBeCalledWithGrainInterface.TestFunc", "Call non orlean grain object", 1000).Wait();
                    test.Call("TestGrain.TestFunc", "Call Test", 100).Wait();
                    Console.WriteLine("Invoke : " + test.Invoke("TestGrain.AnswerFunc", "Invoke Test", 200).Result);
                    Thread.Sleep(3000);
                }
                catch(Exception e)
                {
                    Console.WriteLine(e.ToString());
                }
            }
        }
    }
}

```

## 6 Deploy directory structure and test scripts

directory structure:

![image-20220223182216079](C:\Users\majp\AppData\Roaming\Typora\typora-user-images\image-20220223182216079.png)

scripts:

```shell
# Server side
./svr1/TestOrleans.exe gsvr
./svr2/TestOrleans.exe gsvr
./svr3/TestOrleans.exe gsvr
# Client side
./cli/TestOrleans.exe gcli
```

## 7 test cases

### 7.1 test environment

#### A. Dotnet core 3.14

You can find it in directory: gamejam/environment/dotnetcore31/dotnet-sdk-3.1.416-win-x64.exe, which can be installed in windows x64 os.

#### B. Configure files

I provide three server-side programs. Their configuration files are similar, and the contents are as follows:

```shell
# svr1/data/Config.ini
[Silo]
NodeName = OGSClusterTest1
ClusterId=OGSSvrsCluster
ServiceId=OGSSvrs
SiloPort=33401
GatewayPort=43001
consulurl=http://127.0.0.1:8500/

# svr2/data/Config.ini
[Silo]
NodeName = OGSClusterTest2
ClusterId=OGSSvrsCluster
ServiceId=OGSSvrs
SiloPort=33402
GatewayPort=43002
consulurl=http://127.0.0.1:8500/


# svr3/data/Config.ini
[Silo]
NodeName = OGSClusterTest3
ClusterId=OGSSvrsCluster
ServiceId=OGSSvrs
SiloPort=33403
GatewayPort=43003
consulurl=http://127.0.0.1:8500/

```

#### C. Consul

I use consul as the service configuration center, which can be found in directory: gamejam/environment/consul, and it can be runed by script file start.bat.

![image-20220224094641601](C:\Users\majp\AppData\Roaming\Typora\typora-user-images\image-20220224094641601.png)

### 7.2 test cases

#### A. launch 

launch three server programs

```shell
./svr1/TestOrleans.exe gsvr
./svr2/TestOrleans.exe gsvr
./svr3/TestOrleans.exe gsvr
```

launch a client programs

```shell
./cli/TestOrleans.exe gcli
```

Then, the client will continuously call 100 grain services, which will be created in three server processes in a balanced manner.

#### B. Kill a server process. 

When killed a server process, if the grain service in this process is called by the client again, they will be created by two other server processes. **In this way, the client will not know that the server process is down and the service is uninterrupted.**

#### C. Start the server process again.

Start the server process again. When the client calls the subsequent grain service, **Orleans runtime will forward the new grain service request to the new process**.

### 7.3 Some other examples,

These are also included in the project 'TestOrleans', such as: sub/pub，encryption，etc.

### 7.4 Forecast

The framework is based on DOTNET core which can run on multiple different operating systems such as Windows, Linux, SunOS and so on. Basically, it can solve any cloud computing problems.
