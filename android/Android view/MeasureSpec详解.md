### 参数
MeasureSpec是View中的内部类，基本都是二进制运算。由于int是32位的，用高两位表示mode，低30位表示size，MODE_SHIFT = 30的作用是移位

MODE_MASK = 0x3 << MODE_SHIFT，其中0x3是十六进制，它的二进制是11，向左移位30，结果是11000.....0000（一共30个0），1与任何数“&运算”均为任何数，0与任何数“&运算”均为0，故它的作用是获取measureSpec的高两位的mode，或者低30位的size（MODE_MASK取返，00111...111）

UNSPECIFIED = 0 << MODE_SHIFT，十进制0的二进制是00，左移30位是00000...000（00+30个0）

EXACTLY = 1 << MODE_SHIFT，十进制1的二进制是01，左移30位是00000...000（01+30个0）

AT_MOST = 2 << MODE_SHIFT，十进制2的二进制是10，左移30位是00000...000（10+30个0）

### 方法
makeMeasureSpec将mode、size二进制运算，如size=4（二进制100），mode是EXACTLY（01000...000）,运算结果01000...100

getMode获取measureSpec 中的mode，与MODE_MASK进行“&运算”，如：10000...100与MODE_MASK（11000...000），结果是10000...000，获取mode

getSize获取measureSpec中的size，与MODE_MASK（00111...111）进行“&运算”，如：10000...100与MODE_MASK（00111...111），结果是00000...100，获取size

### 三种模式

1. UNSPECIFIED：不对View大小做限制，如：ListView，ScrollView
2. EXACTLY：确切的大小，如：100dp或者march_parent
3. AT_MOST：大小不可超过某数值，如：wrap_content

### 子View的MeasureSpec

子View的MeasureSpec由父View的MeasureSpec和子View本身的LayoutPramas共同决定

### 总结

不管父View是何模式，若子View有确切数值，则子View大小就是其本身大小，且mode是EXACTLY
若子View是match_parent，则模式与父View相同，且大小同父View（若父View是UNSPECIFIED，则子View大小为0）
若子View是wrap_content，则模式是AT_MOST，大小同父View，表示不可超过父View大小（若父View是UNSPECIFIED，则子View大小为0）
