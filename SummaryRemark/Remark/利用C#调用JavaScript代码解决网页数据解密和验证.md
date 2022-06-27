### 利用C#调用JavaScript代码解决网页数据解密和验证

------

#### 前言

​		一些做数据的公司可能因各种原因，对网络爬虫比较严格，针对网络爬虫施展了五花八门的反爬技术，其中，一般都包含了对请求的验证和对数据的解密和加密，传输数据并不用明文展示。文档展示了个人对破解这些防爬方式的常用手段和经验总结。

#### 破解工具和使用方法

​		一般地，抓取的目标通常为浏览器的网站，推荐使用谷歌浏览器，按f12进入开发者工具，主要用到开发者工具里面的“网络”、"源代码"和"控制台"。

+ 网络
  + 打开网络界面，它能从打开那一刻开始，记录网站发送过的所有请求，刷新页面后，一般先用搜索功能，先输入目标数据的其中一个，查看是否有其中一条请求被匹配，如果数据是明文，则能直接搜索出结果，但数据是加密处理过的，就无法搜索出结果，也可能是蕴含在socket传输当中。
+ 源代码
  + 打开源代码界面，可以用右上角的更多->搜索功能，搜索关键字，关键字可以根据请求名来搜索，也可以用右下方的"xhr断点"打上请求链接的一部分，在可疑的地方打上断点，打完断点之后需要刷新，如果代码执行经过了断点，就会停下，之后就可以慢慢调试，总之可以使用一切可以用方式，在各个地方打上断点，如何慢慢调试，这个就需要长期的使用经验才能熟练。
+ 控制台
  + 控制台一般在源代码打断点的时候，输出当前代码某个变量的运算结果，或者对于某个结果，调用js本身的代码对某个结果加以运算，查看进一步的计算结果。

#### 在C#环境中调用js代码

+ 调试js代码

  + 当断点到可疑代码的时候，需要从输入加密数据的方法开始追踪，一直追踪到输出明文为止，中间的过程即为数据的解密过程。整理好代码之后，可疑用idea创建js项目，并搭建Node.js环境，调试结果。

+ 在c#中调用js代码

  + 在调用js代码前先搭建js代码的执行环境，在c#项目中添加Noesis.Javascript.dll的引用，添加引用后需要将js文件放到bin目录下调用。
  + 调用时需要先读取js文件，存储为string变量，然后交给JavaScriptContext对象调用Run方法执行即可

  ```c#
  		/// <summary>
          /// sign验证，js版本
          /// </summary>
          /// <param name="url"></param>
          /// <returns></returns>
          public static string GetSign_verJs(string url)
          {
              string jsParas = CreateJsParas(url);
              string jsPath = AppDomain.CurrentDomain.BaseDirectory + 							"js\\GetSign.js";
              string jsString = File.ReadAllText(jsPath, 											System.Text.Encoding.UTF8);
              if (string.IsNullOrEmpty(jsString))
              {
                  Console.WriteLine("js文件为空！！");
                  return "";
              }
              jsString += jsParas;
              JavascriptContext context = new JavascriptContext();
              return context.Run(jsString).ToString();
          }
  ```

  + Noesis.JavaScript.dll在c#项目中运行有一个坑，就是涉及到异步，就是两个线程都会使用调用js的时候，会出现内存错误，显示类似“访问不可访问的内存，内存可能损坏”的异常。上网百度也没有处理方法，也没有其他好用的，可以在c#项目里调用js的方法，只能继续使用现在这个dll，需要新建一个项目，这个项目开始是从控制台读一行输入参数，然后将参数拼接到js里面，再执行，输出结果。再编译这个项目，生成exe文件，给原来的项目以调用外部exe的方式使用。

  ```c#
  			string result = "";
              string timestamp = DateTime.Now.ToString("yyyyMMddHHmmssfff") + 					new Random().Next(1000, 10000);//选择时分秒毫秒为内存映射名，保证唯一性
              using (MemoryMappedFile mmf = 													MemoryMappedFile.CreateOrOpen(timestamp, 1024000, 							MemoryMappedFileAccess.ReadWrite))//目前开辟1M内存来处理，如果存在更大的数据量，可扩容
              {
                  ProcessStartInfo p = new ProcessStartInfo("js\\JsDemo.exe", timestamp);
                  p.WindowStyle = ProcessWindowStyle.Normal;
                  p.UseShellExecute = false;
                  p.RedirectStandardOutput = true;
                  p.RedirectStandardInput = true;
                  using (Process process = Process.Start(p))
                  {
                      using (StreamWriter writer = process.StandardInput)
                      {
                          writer.WriteLine(url);
                      }
                      using (StreamReader reader = process.StandardOutput)
                      {
                          result = reader.ReadToEnd();
                          reader.Dispose();
                      }
                  }
                  mmf.Dispose();
              }
              Console.WriteLine(result.Trim());
              return result.Trim();
  ```

  + 另外，如果觉得这种方法效率太低，也可以辛苦自己，照搬js的代码，自己动手写成c#的代码。



