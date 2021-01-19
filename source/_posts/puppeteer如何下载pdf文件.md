title:  puppeteer如何下载pdf文件
date: 2021-01-19 09:37:00
tags:
- chrome
- devtools
- pdf
- puppeteer

categories: nodejs
---

相信大家伙儿都清楚，访问一些常见的文件格式，浏览器是会“自作主张”的提供预览功能的，但有时候我们想直接下载到文件。
尤其是在用puppeteer这种方式进行操作时，我们如何绕过浏览器的默认行为呢？

拿PDF文件来举个例子吧，因为chrome默认会使用内置的pdf viewer来直接预览文件，而浏览器进入到预览模式后其实上下文发生了切换，导致无法像操作普通页面那样来操作页面上的元素。
此时，我们就需要必须要找到办法直接下载文件了。

目前要下载文件，必须保证程序不是在headless模式下，换句话说即必须要有可视化窗口。接下来，我们就要动手绕过浏览器默认行为了。
思路很简单，浏览器之所以会进行预览，是因为请求的响应头中，包含了浏览器适配预览的内容类型（content-type）标识。所以我们的目标就是要想办法覆盖这个响应头设置即可。

查阅了一下文章后，发现chrome确实提供了这种接口：

```javascript
......
    const browser = await puppeteer.launch({
        headless: false, 
        timeout: 60000, 
    });
    const page = await browser.newPage();
    const client = await page.target().createCDPSession();  // 激活 chrome devtools模式
    await client.send('Fetch.enable', { // 开启request domain的控制开关
        patterns: [
            {
                urlPattern: '*',    // 设置视图控制的request url模式
                requestStage: 'Response', // 设置 目标请求的控制阶段， Response表示为拿到服务器响应内容，并在交给浏览器流程之前的阶段
            }
        ]
    });

    // 所有匹配url模式的请求都会触发这个事件回调，再没有手动触发Fetch.fulfillRequest，continueRequest, failRequest等事件之前，请求处于暂停状态
    await client.on('Fetch.requestPaused', async (reqEvent)=>{
        const {requestId} = reqEvent;   // 获取浏览器为当前请求分配的编号

        // 检查默认的responseHeader中是否包含目标内容
        let responseHeaders = reqEvent.responseHeaders || [];
        let contentType = '';

        for(let elements of responseHeaders){
            if(elements.name.toLowerCase() == 'content-type'){
                contentType = elements.value;
            }
        }

        if(contentType.endsWith('pdf')){
            responseHeaders.push({  // 直接覆盖默认的格式为附件类型，避免触发浏览器的默认预览模式
                name: 'content-disposition',
                value: 'attachment',
            });

            const responseObj = await client.send('Fetch.getResponseBody', {requestId});    // 获取原本请求的响应内容

            await client.send('Fetch.fulfillRequest', { // 覆盖默认的响应数据属性
                requestId,
                responseCode: 200,
                responseHeaders,
                body: responseObj.body,
            });
        }else{
            await client.send('Fetch.continueRequest', {requestId});    // 直接“放行”，不进行任何属性覆盖
        }
    });

    try{
        await client.send('Page.setDownloadBehavior', {behavior: 'allow', downloadPath: "这里填写想要保存附件的位置，默认会下载到系统的downloads文件夹"});
        await page.goto('这里填写pdf目标链接');
    }catch(err){
        console.log(err);   // 目前会抛出一个异常，忽略即可。
    }

    await client.send('Fetch.disable'); // 关闭 request domain的控制开关
    await client.detach(); // 关闭cdpSession
......

```

这样处理后，chrome就不会做任何“多余”的操作，直接老老实实的把文件下载到本地了。

不过需要注意的是，下面参考资料中，把这个方案用到的一些方法表示为：实验性（Experimental），已弃用（Deprecated）。
不排除未来的新版本chrome不再支持这些接口了。


希望这篇文章帮到你了，回见~~

### 参考资料

[stack overflow](https://stackoverflow.com/questions/56254177/open-puppeteer-with-specific-configuration-download-pdf-instead-of-pdf-viewer)
[chrome devtools protocol](https://chromedevtools.github.io/devtools-protocol/tot/Fetch/)
[CDPSession](https://pptr.dev/#?product=Puppeteer&version=v5.5.0&show=api-class-cdpsession)