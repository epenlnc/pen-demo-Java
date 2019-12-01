# 1.收到服务端发过来的包
# 2.判断数据是否合法
# 3.判断通信，根据协议中的每个不同包的命令进行判断
# 4.解包(此处只展示点数据包的解析过程)
# 
```java
    private void pen(String currentData) {
        //连接包最小位数为40位
        if (currentData.length() < 94) {
            cache.put(ctx.channel().id(), currentData);
            return;
        }
        // 协议中笔迹长度
        int dataSize = Integer.parseInt(currentData.substring(38, 42), 16) * 2;
        // 协议中笔迹
        String data = currentData.substring(42);
        if (data.length() < dataSize) {
            cache.put(ctx.channel().id(), currentData);
            return;
        }
        // 协议中的笔迹点数据
        String dataPoints = currentData.substring(42, 42 + dataSize);
        // 协议中的笔迹点数据个数
        int pointNum = Integer.parseInt(dataPoints.substring(16, 18), 16);
        String point = dataPoints.substring(18);
        if (point.length() < 34 * pointNum) {
            // TODO 包是不完整的，在此处存储起来，下一次通信时拼接上
            return;
        }
        //包中的一个点
        String firstDot = currentData.substring(0, 42 + dataSize);
    }
    服务端通信一次传过来的可能不止一个包，所以在解析完一个完整包以后，还需要判断是否此次通信还有数据，如果还有继续从第二步开始
```
# 5.一个完整的点从包中取出来，继续进行下一步
```java
    /**
     * 16进制字符串转换成为点对象集合
     *
     * @param dotData
     * @param mac
     * @return
     */
    public static List<Dot> HexToDot(String dotData, String mac) {
        SendDot sendDot = new SendDot();
        int byteSize = 2;
    
        sendDot.setBookId(Long.parseLong(dotData.substring(byteSize, byteSize += 4), 16));
        sendDot.setOwnerId(Long.parseLong(dotData.substring(byteSize, byteSize += 4), 16));
        sendDot.setSectionId(Long.parseLong(dotData.substring(byteSize, byteSize += 4), 16));
        sendDot.setPageId(Long.parseLong(dotData.substring(byteSize, byteSize += 2), 16));
        int pointSize = Integer.parseInt(dotData.substring(byteSize, byteSize + 2), 16);
    
        List<Point> pointList = new ArrayList<>();
        String pointString = dotData.substring(18);
        byteSize = 0;
        Point point;
        for (int i = 0; i < pointSize; i++) {
            point = new Point();
            int type = Integer.parseInt(pointString.substring(byteSize, byteSize += 2), 16);
            switch (type) {
                case 0:
                    point.setDotType("PEN_DOWN");
                    break;
                case 1:
                    point.setDotType("PEN_MOVE");
                    break;
                case 2:
                    point.setDotType("PEN_UP");
                    break;
                default:
                    break;
            }
            point.setForce(Long.parseLong(pointString.substring(byteSize, byteSize += 4), 16));
            point.setX(Long.parseLong(pointString.substring(byteSize, byteSize += 4), 16));
            point.setY(Long.parseLong(pointString.substring(byteSize, byteSize += 4), 16));
            point.setFx(Long.parseLong(pointString.substring(byteSize, byteSize += 2), 16));
            point.setFy(Long.parseLong(pointString.substring(byteSize, byteSize += 2), 16));
            point.setTimestamp(Long.parseLong(pointString.substring(byteSize, byteSize += 16), 16));
            pointList.add(point);
        }
        sendDot.setPointList(pointList);
        dotSize += pointList.size();
        return SendDotToDots.convert(sendDot, mac);
    }
```
#转成点对象集合
```java
    public static List<Dot> convert(SendDot sendDot, String mac) {
        List<Dot> dots = new ArrayList<>();
        for (Point point : sendDot.getPointList()) {
            Dot dot = new Dot();
            BeanUtils.copyProperties(point,dot);
            dot.setAbX(point.getX() + point.getFx() / 100F);
            dot.setAbY(point.getY() + point.getFy() / 100F);
            dot.setTimeLong(point.getTimestamp());
            dot.setOwnerID(sendDot.getOwnerId());
            dot.setSectionID(sendDot.getSectionId());
            dot.setBookID(sendDot.getBookId());
            dot.setPageID(sendDot.getPageId());
            dot.setMac(mac);
            dots.add(dot);
        }
        return dots;
    }
```    
# 
```java
    @Data
    public class SendDot {
        private Long bookId;
        private Long sectionId;
        private Long ownerId;
        private Long pageId;
        private List<Point> pointList;
    }
    
    @Data
    class Point {
        private String dotType;
        private Long force;
        private Long x;
        private Long y;
        private Long fx;
        private Long fy;
        private Long timestamp;
    }
```
#点对象
```java
        @Data
        public class Dot {
        
            private String mac;
            /**
             * 点计数
             */
            private Long counter;
            /**
             * 区域ID
             */
            private Long sectionID;
            
            private Long ownerID;
            /**
             * 书号
             */
            private Long bookID;
            /**
             * 页号
             */
            private Long pageID;
            /**
             * 当前点的RTC 时间，返回时间戳 ms（起止时间是2010-01-01 00:00:00,000）
             */
            private Long timeLong;
            /**
             * 点横坐标，整数部分
             */
            private Long x;
            /**
             * 点纵坐标，整数部分
             */
            private Long y;
            /**
             * 点横坐标，小数部分
             */
            private Float fx;
            /**
             * 点纵坐标，小数部分
             */
            private Float fy;
            /**
             * 点横坐标，整数+小数部分
             */
            private Float abX;
            /**
             * 点纵坐标，整数+小数部分
             */
            private Float abY;
            /**
             * 类型（1：笔按下去，0：笔抬起来，2：笔在移动）
             */
            private String dotType;
            /**
             * 点的角度值
             */
            private Long angle;
            /**
             * 笔的颜色值
             */
            private Long color;
            /**
             * 点的压力值
             */
            private Long force;
        }
```

# 此处只展示连接包的组装
```java
        /**
         * 连接数据转16进制数据
         *
         * @param cid           定义一个唯一不重复的值
         * @param uidList       笔盒uid集合
         */
        public static byte[] connectionToHex(long cid, List<Long> uidList) {
            StringBuilder stringBuilder = new StringBuilder();
            //req
            stringBuilder.append("00000001");
            //cid
            stringBuilder.append(String.format("%016x", cid));
            // data_type
            stringBuilder.append("0011");
            // base_ts   固定 000001775B094836
            stringBuilder.append("000001775B094836");
            //uid_size
            stringBuilder.append(String.format("%04x", uidList.size()));
            //uid_s
            for (Long uid : uidList) {
                stringBuilder.append(String.format("%016x", uid));
            }
            return hexStringToByteArray(stringBuilder.toString().toUpperCase());
        }
```

#通信使用byte数组
```java
    @Override
    protected void initChannel(SocketChannel channel) throws Exception {
        channel.pipeline().addLast("encoder", new ByteArrayEncoder());
        channel.pipeline().addLast("decoder", new ByteArrayDecoder());
        channel.pipeline().addLast(clientChannelHandler);
    }
```
