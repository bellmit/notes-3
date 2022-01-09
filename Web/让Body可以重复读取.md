# 概述

众所周知，ServletHttpRequest 的请求体数据只能读取一次，网上有众多解决可以重复读取的例子，但是！！！几乎都没有解决 `multipart/form-data` 格式的请求，比如使用表单上传文件。

# fix it

其实很简单，使用在他们的基础上，重写 `getParts` 方法即可。代码如下：

```java
import javax.servlet.ReadListener;
import javax.servlet.ServletException;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import javax.servlet.http.Part;
import java.io.*;
import java.util.Collection;
import java.util.Collections;

/**
 * 支持可重复读的请求体
 *
 * @author WuQinglong
 */
public class RequestBodyRepeatableReadWrapper extends HttpServletRequestWrapper {

    /**
     * 支持 form-data
     */
    private Collection<Part> parts;

    /**
     * 请求体数据
     */
    private final ByteArrayOutputStream bodyBytes;

    public RequestBodyRepeatableReadWrapper(HttpServletRequest request) {
        super(request);

        // form-data
        try {
            parts = request.getParts();
        } catch (IOException | ServletException e) {
            e.printStackTrace();
            parts = Collections.emptyList();
        }

        // 请求体数据
        bodyBytes = new ByteArrayOutputStream();
        try (InputStream inputStream = request.getInputStream()) {
            int hasRead;
            byte[] buffer = new byte[4096];
            while ((hasRead = inputStream.read(buffer)) != -1) {
                bodyBytes.write(buffer, 0, hasRead);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Collection<Part> getParts() {
        return parts;
    }

    @Override
    public ServletInputStream getInputStream() {
        return new ServletInputStream() {
            private final byte[] body = bodyBytes.toByteArray();
            private int index = 0;
            private ReadListener readListener;

            @Override
            public boolean isFinished() {
                return index >= body.length;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {
                this.readListener = readListener;
            }

            public int read() {
                try {
                    if (!isFinished()) {
                        return body[index++];
                    }
                    readListener.onAllDataRead();
                } catch (Throwable e) {
                    readListener.onError(e);
                }
                return -1;
            }
        };
    }

    @Override
    public BufferedReader getReader() {
        return new BufferedReader(new InputStreamReader(this.getInputStream()));
    }

    /**
     * 获取请求体
     */
    public byte[] getBody() {
        return bodyBytes.toByteArray();
    }
}
```

**注意**：这种方式仅适用于使用 `StandardServletMultipartResolver` 进行文件的解析，不适用于使用 commons-fileupload 包进行文件的解析。
