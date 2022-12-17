# C++日志系统设计与实现

- 日志存储：文本文件
- 日志内容：时间-级别-文件-行号-内容 
- 日志级别：debug-info-warn-error-fatal，设计对应的宏
- 日志翻滚：设置日志的大小，超出设置的大小则备份
- 采用C++单例模式设计

缺陷：线程不安全

Log类设计如下：
```cpp
class Log {
 public:
  enum Level { DEBUG = 0, INFO, WARN, ERROR, FATAL, LEVEL_COUNT };
  static Log* instance();
  void open(const std::string& filename);
  void close();
  // 日志内容记录
  void log(Level level, const char* file, int line, const char* format, ...); 
  void max(int bytes);
  void level(Level level);

 private:
  Log();
  ~Log();
  void rotate(); //日志翻滚

 private:
  std::string m_filename;
  std::ofstream m_fout;
  static const char* s_level[LEVEL_COUNT];
  static Log* m_instance;
  int m_max;
  Level m_level;
  int m_len;
};
```

宏设计：
```cpp
#define debug(format, ...) \
  Log::instance()->log(Log::DEBUG, __FILE__, __LINE__, format, ##__VA_ARGS__)
```