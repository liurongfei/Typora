- 按照数据流向，可以将流分为输入流和输出流，其中输入流只能读取数据、不能写入数据，而输出流只能写入数据、不能读取数据。
- 按照数据类型，可以将流分为字节流和字符流，其中字节流操作的数据单元是8位的字节，而字符流操作的数据单元是16位的字符。
- 按照处理功能，可以将流分为节点流和处理流，其中节点流可以直接从/向一个特定的IO设备（磁盘、网络等）读/写数据，也称为低级流，而处理流是对节点流的连接或封装，用于简化数据读/写功能或提高效率，也称为高级流。