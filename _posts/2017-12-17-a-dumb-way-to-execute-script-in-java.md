---
layout: post
title: 一种在Java中执行脚本的方法
date: 2017-12-17 20:00:00 +0800
tags:
- process
- RandomAccessFile
---

本文实现了一种在 web 前端点击按钮来触发执行预先写好的 shell 脚本(不限于shell)，然后将脚本执行结果返回前端进行显示的方法。

总的思路大致为:

1. 点击前端按钮，命令由后端的 Controller 接收
2. Controller 调用某个 Service 来执行 Shell 脚本，并将结果存入一个临时文件
3. 将临时文件名称返回给前端
4. 前端通过定时请求将文件的内容显示到前端

参照 [When Runtime.exec() won't](https://www.javaworld.com/article/2071275/core-java/when-runtime-exec---won-t.html)  中的例子写了第一版:

```java
package com.zhjwpku.util

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.*;

class StreamGobbler extends Thread {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    InputStream is;
    String type;
    OutputStream os;

    StreamGobbler(InputStream is, String type) {
        this(is, type, null);
    }

    StreamGobbler(InputStream is, String type, OutputStream redirect) {
        this.is = is;
        this.type = type;
        this.os = redirect;
    }

    @Override
    public void run() {
        BufferedReader reader = null;
        BufferedWriter writer = null;
        try {
            reader = new BufferedReader(new InputStreamReader(is));
            if (os != null) {
                writer = new BufferedWriter(new OutputStreamWriter(os));
            }

            String line = null;
            while ((line = reader.readLine()) != null) {
                if (writer != null) {
                    writer.append(line).append("\n");
                    writer.flush();
                } else {
                    System.out.println(type + "> " + line);
                }
            }
        } catch (IOException e) {
            logger.error("Handle stream error.", e);
        } finally {
            try {
                if (reader != null) reader.close();
            } catch (IOException e) {
                logger.error("close reader stream error.", e);
            }
            try {
                if (writer != null) writer.close();
            } catch (IOException e) {
                logger.error("close writer stream error.", e);
            }
        }
    }
}

public class CMDUtil {
    private static Logger logger = LoggerFactory.getLogger(CMDUtil.class);

    public static void ExecuteCmd(String command, String ofilename) {
        try {
            FileOutputStream fos = new FileOutputStream(ofilename);
            Process p = Runtime.getRuntime().exec(command);

            // 需要读取输入流缓冲区和错误流缓冲区
            StreamGobbler errorGobbler = new StreamGobbler(p.getErrorStream(), "ERROR");
            StreamGobbler outputGobbler = new StreamGobbler(p.getInputStream(), "OUTPUT", fos);

            errorGobbler.start();
            outputGobbler.start();

            int exitVal = p.waitFor();
            logger.info("ExitValue: " + exitVal);
        } catch (Exception e) {
            logger.error("execute shell error: ", e);
        }
    }
}
```

上面的程序在不涉及网页的情况下能很好的工作，但是偏偏这个需求是需要从网页调用，来看一下网页通过 Controller 调取后端 Service 的代码:

```java
import com.zhjwpku.dto.ReadFileResult;
import com.zhjwpku.service.ShellCommandTest;
import com.zhjwpku.util.CMDUtil;
import com.zhjwpku.util.RandomAccessFileUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.io.*;
import java.net.URL;

@Service
public class ShellCommandTestImpl implements ShellCommandTest {
    protected Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public String ping(Integer count) {
        URL url = getClass().getClassLoader().getResource("ping.sh");

        if (url == null) {
            url = getClass().getClassLoader().getResource("shell/ping.sh");
        }

        File file = new File(url.getFile());

        if (count <= 0) count = 4;

        String command = file.getPath() + " " + count;
        String ofilename = "/tmp/ping-" + System.currentTimeMillis() + ".txt";

        CMDUtil.ExecuteCmd(command, ofilename);

        return ofilename;
    }

    @Override
    public ReadFileResult readFile(String filePath, Long position) {
        logger.info("filePath = {}, position = {}", filePath, position);

        ReadFileResult result = RandomAccessFileUtils.readFromFile(filePath, position);

        return result;
    }
}
```

上面的代码在执行 CMDUtil.ExecuteCmd(command, ofilename) 时由于 `p.waitFor()` 会在进程没有结束的时候阻塞，导致文件的名称不能返回给前端。一个解决的办法是不调用 `p.waitFor()`，即让子进程自生自灭。另一种方式是借助线程:

*CMDUtil*

```java
package com.zhjwpku.util;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.*;

class StreamGobbler extends Thread {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    InputStream is;
    String type;
    OutputStream os;

    StreamGobbler(InputStream is, String type) {
        this(is, type, null);
    }

    StreamGobbler(InputStream is, String type, OutputStream redirect) {
        this.is = is;
        this.type = type;
        this.os = redirect;
    }

    @Override
    public void run() {
        BufferedReader reader = null;
        BufferedWriter writer = null;
        try {
            reader = new BufferedReader(new InputStreamReader(is));
            if (os != null) {
                writer = new BufferedWriter(new OutputStreamWriter(os));
            }

            String line = null;
            while ((line = reader.readLine()) != null) {
                if (writer != null) {
                    writer.append(line).append("\n");
                    writer.flush();
                } else {
                    System.out.println(type + "> " + line);
                }
            }
        } catch (IOException e) {
            logger.error("Handle stream error.", e);
        } finally {
            try {
                if (reader != null) reader.close();
            } catch (IOException e) {
                logger.error("close reader stream error.", e);
            }
            try {
                if (writer != null) writer.close();
            } catch (IOException e) {
                logger.error("close writer stream error.", e);
            }
        }
    }
}

public class CMDUtil extends Thread {
    private static Logger logger = LoggerFactory.getLogger(CMDUtil.class);

    private String command;
    private String ofilename;

    public CMDUtil(String command, String ofilename) {
        this.command = command;
        this.ofilename = ofilename;
    }

    @Override
    public void run() {
        try {
            FileOutputStream fos = new FileOutputStream(ofilename);
            Process p = Runtime.getRuntime().exec(command);

            StreamGobbler errorGobbler = new StreamGobbler(p.getErrorStream(), "ERROR");
            StreamGobbler outputGobbler = new StreamGobbler(p.getInputStream(), "OUTPUT", fos);

            errorGobbler.start();
            outputGobbler.start();

            int exitVal = p.waitFor();
            logger.info("ExitValue: " + exitVal);
        } catch (Exception e) {
            logger.error("execute shell error: ", e);
        }
    }
}
```

*ShellCommandTestImpl*

```java
import com.zhjwpku.dto.ReadFileResult;
import com.zhjwpku.service.ShellCommandTest;
import com.zhjwpku.util.CMDUtil;
import com.zhjwpku.util.RandomAccessFileUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.io.*;
import java.net.URL;

@Service
public class ShellCommandTestImpl implements ShellCommandTest {
    protected Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public String ping(Integer count) {
        URL url = getClass().getClassLoader().getResource("ping.sh");

        if (url == null) {
            url = getClass().getClassLoader().getResource("shell/ping.sh");
        }

        File file = new File(url.getFile());

        if (count <= 0) count = 4;

        String command = file.getPath() + " " + count;
        String ofilename = "/tmp/ping-" + System.currentTimeMillis() + ".txt";

        Thread cmd = new CMDUtil(command, ofilename);
        cmd.start();

        return ofilename;
    }

    @Override
    public ReadFileResult readFile(String filePath, Long position) {
        logger.info("filePath = {}, position = {}", filePath, position);

        ReadFileResult result = RandomAccessFileUtils.readFromFile(filePath, position);

        return result;
    }
}
```

上面代码实现了用户请求后调用脚本并返回结果文件名的操作。在前端需要定时请求以获取结果文件的内容。代码如下:

```javascript
/**
 * Created by zhjwpku on 2017/11/24.
 */

$(function () {
    $('.ajax-loading-animation').hide();
    $('.box').show();

    $.ajax({
        type: "GET",
        async: false,
        url: "/command/ping",       // 执行ping命令
        data: {"count": "10"},
        dataType: "json",
        headers: createAuthorizationTokenHeader(),
        error: function () {
        },
        success: function (data) {
            if (data && data.code == 200) {
                window.setTimeout(getResult, 1000, data.file_path, 0);
            }
        }
    });

    function getResult(file_path, position) {
        $.ajax({
            type: "GET",
            async: false,
            url: "/result/ping?file_path=" + file_path + "&position=" + position,
            dataType: "json",
            headers: createAuthorizationTokenHeader(),
            error: function () {
            },
            success: function (data) {
                if (data && data.code == 200) {
                    console.log(data.result);
                    if (data.result.content != null) {
                        $("#result").append(data.result.content.replace(/\r?\n/g, "<br />"));
                    }
                    window.setTimeout(getResult, 2000, file_path, data.result.position);
                }
            }
        });
    }
});
```

负责根据上次读取的位置来进行本次请求，随机读取文件的实现如下:

```java
package com.zhjwpku.util;

import com.zhjwpku.dto.ReadFileResult;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.RandomAccessFile;

public class RandomAccessFileUtils {

    private static final Logger logger = LoggerFactory.getLogger(RandomAccessFileUtils.class);

    public static ReadFileResult readFromFile(String filePath, Long position) {

        ReadFileResult result = new ReadFileResult();
        RandomAccessFile file = null;

        try {
            file = new RandomAccessFile(filePath, "r");
            Long fileLength = file.length();

            if (position >= fileLength) {       // 要读取的位置超出了文件的大小
                result.setPosition(fileLength);
                result.setContent(null);
            } else {
                file.seek(position);
                int size = (int)(fileLength - position);
                byte[] bytes = new byte[size];

                file.read(bytes);

                result.setPosition(fileLength);
                result.setContent(new String(bytes));
            }
        } catch (Exception e) {
            logger.error("execute shell error: ", e);
        } finally {
            if (file != null) {
                try {
                    file.close();
                } catch (IOException e) {
                    logger.error("close file error: ", e);
                }
            }
        }

        return result;
    }
}
```

贴一张最终的效果图(比较糙，求轻喷):

![java_cmd](/assets/201712/java_cmd.gif)

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [When Runtime.exec() won't](https://www.javaworld.com/article/2071275/core-java/when-runtime-exec---won-t.html)<br>
2 [Process.waitFor(), threads, and InputStreams](https://stackoverflow.com/questions/2150723/process-waitfor-threads-and-inputstreams)
</span>
