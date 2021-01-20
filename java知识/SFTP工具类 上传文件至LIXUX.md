# SFTP工具类 上传文件至LIXUX

**最主要是创建目录的逻辑处理**

```java
/**用jsch实现SFTP协议传输文件
 * SFTP工具类
 *
 * @author by cyf
 * @date 2020/9/14.
 */

@Slf4j
@Getter
@Setter
public class SFTPUtils {

    /**
     * 服务器连接ip
     */
    private String host;
    /**
     * 用户名
     */
    private String username;
    /**
     * 密码
     */
    private String password;
    /**
     * 端口号
     */
    private int port;

    private ChannelSftp channel = null;
    private Session sshSession = null;

    public SFTPUtils(String host, String username, String password, int port) {
        this.host = host;
        this.username = username;
        this.password = password;
        this.port = port;
    }


    /**
     * 通过SFTP连接服务器
     */
    public void connect() {
        try {
            JSch jSch = new JSch();
            sshSession = jSch.getSession(username, host, port);
            log.info("Session created");
            sshSession.setPassword(password);
            Properties sshConfig = new Properties();
            sshConfig.put("StrictHostKeyChecking", "no");
            sshSession.setConfig(sshConfig);
            sshSession.connect();
            log.debug("Session connected");
            channel = (ChannelSftp) sshSession.openChannel("sftp");
            //建立SFTP连接
            channel.connect();
        } catch (Exception e) {
            log.warn("connect sftp error:{}", e.getMessage());
        }
    }

    /**
     * 关闭SFTP连接
     */
    public void disconnect() {
        if (this.channel != null) {
            channel.disconnect();
            log.debug("channel is closed");
        }
        if (this.sshSession != null) {
            sshSession.disconnect();
            log.debug("sshSession is closed");
        }
    }

    /**
     * 上传单个文件
     *
     * @param remotePath 远程保持的目录
     * @param fileName   保存文件名
     * @param localPath  本地上传目录(以路径符号结束)
     * @param del        是否删除本地文件
     * @return
     */
    public boolean uploadFile(String remotePath, String fileName, String localPath, boolean del) {
        FileInputStream in = null;
        try {
            //创建文件夹
            createDir(remotePath);
            File file = new File(localPath + fileName);
            in = new FileInputStream(file);
            channel.put(in, fileName);
            return true;
        } catch (FileNotFoundException e) {
            log.warn("upload file not fount :{}", e.getMessage());
        } catch (SftpException e) {
            log.warn("SftpException :{}", e.getMessage());
        } finally {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException e) {
                    log.warn("IOException:{}", e.getMessage());
                }
            }
            //删除文件
            if (del) {
                deleteFile(localPath + fileName);
            }
        }
        return false;
    }

    /**
     * 删除本地文件
     *
     * @param filePath
     */
    public boolean deleteFile(String filePath) {
        File file = new File(filePath);
        if (!file.exists() || !file.isFile()) {
            return false;
        } else {
            boolean result = file.delete();
            if (result) {
                log.info("delete local file success ,file path is:" + filePath);
            }
            return result;
        }
    }

/**
 * 创建远程目录
 *
 * @param createPath 远程文件夹路径
 * @return
 */
public boolean createDir(String createPath) {
    try {
        //判断目录是否存在
        if (isDirExist(createPath)) {
            this.channel.cd(createPath);
            return true;
        } else {
            String[] pathArray = createPath.split("/");
            StringBuffer filePath = new StringBuffer("/");
            for (String path : pathArray) {
                if ("".equals(path)) {
                    continue;
                }
                filePath.append(path + "/");
                if (isDirExist(filePath.toString())) {
                    channel.cd(filePath.toString());
                } else {
                    //建立目录
                    channel.mkdir(filePath.toString());
                    log.info("create directory :{}",filePath);
                    //进入当前目录
                    channel.cd(filePath.toString());
                }
            }
            this.channel.cd(createPath);
            return true;
        }
    } catch (SftpException e) {
        log.warn("create remote dir error :{}", e.getMessage());
    }
    return false;
}

/**
 * 判断目录是否存在
 *
 * @param directory
 * @return
 */
public boolean isDirExist(String directory) {
    boolean isDirExistFlag = false;
    try {
        SftpATTRS sftpATTRS = channel.lstat(directory);
        isDirExistFlag = true;
        return sftpATTRS.isDir();
    } catch (Exception e) {
        log.debug("the directory is not exist : {}", directory);
    }
    return isDirExistFlag;
}
```

