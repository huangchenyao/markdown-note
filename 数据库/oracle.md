## 数据类型
### 数据类型概述
Oracle提供了22种不同的数据类型，简要地讲，有以下几种数据类型。  
- CHAR：定长字符串，空格填充，最多2000字节
- NCHAR：UNICODE编码，定长字符串，空格填充，最多2000字节
- VARCHAR2：是VARCHAR的同义词，变长字符串，最多4000字节
- NVARCHAR2：UNICODE编码，变长字符串，最多4000字节
- RAW：变长二进制，最多2000字节
- NUMBER：最多38位的数字
- BINARY_FLOAT：32位单精度浮点数
- BINARY_DOUBLE：64位双精度浮点数
- LONG：最多2GB（2GBytes）的字符数据，建议转CLOB
- LONG RAW：最多2GB（2GBytes）的二进制数据，建议转BLOB
- DATE：年月日时分秒，定宽7字节
- TIMESTAMP：年月日时分秒.小数秒，定宽7/11字节
- TIMESTAMP WITH TIME ZONE：带时区信息，定宽13字节
- TIMESTAMP WITH LOCAL TIME ZONE：参考数据库时区信息，定宽7/11字节
- INTERVAL YEAR TO MONTH：时段，年数和月数，定宽5字节
- INTERVAL DAY TO SECOND：时段，天/小时/分钟/秒数（小数秒），定宽11字节
- BFILE：目录对象
- BLOB：二进制数据，4GB*10（数据库块大小）字节
- CLOB：纯文本，4GB*10（数据库块大小）字节
- NCLOB：纯文本，用数据库国家字符集编码，4GB*10（数据库块大小）字节
- ROWID
- UROWID

### 字符和二进制串类型
- VARCHAR2(<SIZE> <BYTE|CHAR>)：SIZE 1~4000
- CHAR(<SIZE> <BYTE|CHAR)：SIZE 1~2000
- NVARCHAR2(<SIZE>)：SIZE大于0，上界由国家字符集指定
- NCHAR(<SIZE>)：SIZE大于0，上界由国家字符集指定
- varchar和nvarchar的区别：
    - varchar(4) 可以输入4个字符，也可以输入两个汉字  
    - nvarchar(4) 可以输四个汉字，也可以输4个字母，但最多四个  

### 二进制串：RAW类型
- RAW
- LONG
- LONG RAW

### 数值类型
- NUMBER(p, s)：精度可达38位，范围10^-130^~10^126^，p为精度（默认38），s为小数位数（默认为0）
    - NUMBERIC(p, s)：完全映射至NUMBER(p, s)
    - DECIMAL(p, s)或DEC(p, s)：完全映射至NUMBER(p, s)
    - INTEGER或INT：完全映射至NUMBER(38)
    - SMALLINT：完全映射至NUMBER(38)
    - FLOAT(p)：完全映射至NUMBER
    - DOUBLE PRECSION：完全映射至NUMBER
    - REAL：完全映射至NUMBER
- BINARY_FLOAT：6位精度，范围±10^38.53^
- BINARY_DOUBLE：13位精度，范围±10^308.25^

### LONG类型


### DATE、TIMESTAMP和INTERVAL类型


### LOB类型（large object，大对象）
- CLOB：字符LOB
- NCLOB：另一种字符LOB
- BLOB：二进制LOB
- BFILE：二进制文件LOB

### ROWID/UROWID类型