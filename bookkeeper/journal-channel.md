[toc]

#### Apache BookKeeper的 基础类库 - BufferedChannel

###### BufferedChannel简介
* BufferedChannel 封装了 `FileChannel`, 在将数据写入`FileChannel`前增加了一个`ByteBuf`的缓存，相应的读取时也需要根据读取开始的位置决定是从文件(`FileChnannel`)中读取，还是从`ByteBuf`中读取;

###### BufferedChannel类继承
我们直接用图来表示
![bufferedChannel](file:///home/lw/文档/apache bookkeeper/bufferedchannel.png)

###### BufferedChannelBase
* 简单封装了 `FileChannel`, 是个抽象类, 用于被其他类继承， 添加不同的功能到 `FileChannel`上
```
public abstract class BufferedChannelBase {
    protected final FileChannel fileChannel;

    protected BufferedChannelBase(FileChannel fc) {
        this.fileChannel = fc;
    }

    // 先判断fileChannel是否已经打开，确保只返回已打开的fileChannel
    protected FileChannel validateAndGetFileChannel() throws IOException {
        if (!fileChannel.isOpen()) {
            throw new BufferedChannelClosedException();
        }
        return fileChannel;
    }
	
    public long size() throws IOException {
        return validateAndGetFileChannel().size();
    }
}
```

###### BufferedReadChannel
* 添加从`FileChennel`中读到数据到`ByteBuf`的功能
* 构造函数
```
public BufferedReadChannel(FileChannel fileChannel, int readCapacity) {
        super(fileChannel);
        this.readCapacity = readCapacity;
		// 初始化这个readBuffer, 用于从FileChannel中读取数据
        this.readBuffer = Unpooled.buffer(readCapacity);
    }
```
* 读取数据
```
public synchronized int read(ByteBuf dest, long pos, int length) throws IOException {
        invocationCount++;
        long currentPosition = pos;
        long eof = validateAndGetFileChannel().size();
        if (pos >= eof) {
            return -1;
        }
		
		// 循环读，直到读取的数据长度达到length
        while (length > 0) {
		
		    //先判断readBuf中有无数据，如果没有的话，从FileChannel中读取
            if (readBufferStartPosition <= currentPosition
                    && currentPosition < readBufferStartPosition + readBuffer.readableBytes()) {
                int posInBuffer = (int) (currentPosition - readBufferStartPosition);
                int bytesToCopy = Math.min(length, readBuffer.readableBytes() - posInBuffer);
                dest.writeBytes(readBuffer, posInBuffer, bytesToCopy);
                currentPosition += bytesToCopy;
                length -= bytesToCopy;
                cacheHitCount++;
            } else if (currentPosition >= eof) {
                break;
            } else {
			    // 从 FileChannel中读取数据到readBuffer中
                readBufferStartPosition = currentPosition;
                int readBytes = 0;
                if ((readBytes = validateAndGetFileChannel().read(readBuffer.internalNioBuffer(0, readCapacity),
                        currentPosition)) <= 0) {
                    throw new IOException("..");
                }
                readBuffer.writerIndex(readBytes);
            }
        }
        return (int) (currentPosition - pos);
    }
```

###### BufferedChannel
* 继承自 `BufferedReadChannel`, 提供了基于缓存的写入功能, 写入的数据先置于`wirteBuffer`中，但其size达到设置的阈值后，会被写入到`FileChannel`中，进而会被sync到磁盘;
* 写入数据
```
public void write(ByteBuf src) throws IOException {
        int copied = 0;
        boolean shouldForceWrite = false;
        synchronized (this) {
            int len = src.readableBytes();
			// 循环写数据到writeBuffer, 如果其间writeBuffer被写满，则flush到FileChannel
            while (copied < len) {
                int bytesToCopy = Math.min(src.readableBytes() - copied, writeBuffer.writableBytes());
                writeBuffer.writeBytes(src, src.readerIndex() + copied, bytesToCopy);
                copied += bytesToCopy;

                if (!writeBuffer.isWritable()) {
                    flush();
                }
            }
            position.addAndGet(copied);
            unpersistedBytes.addAndGet(copied);
            if (doRegularFlushes) {
			    // 判断写入数据量是否达到阈值，达到了则flush
                if (unpersistedBytes.get() >= unpersistedBytesBound) {
                    flush();
                    shouldForceWrite = true;
                }
            }
        }
        if (shouldForceWrite) {
		// sync到磁盘
            forceWrite(false);
        }
    }
```
* 调用了上面的`write`方法后，写入到当前`BufferedChannel`中的数据可能存在于`writeBuffer`和`fileChannel`两部分中,读取时也需要考虑这两部分
* 读取数据
```
public synchronized int read(ByteBuf dest, long pos, int length) throws IOException {
        long prevPos = pos;
        while (length > 0) {
            // check if it is in the write buffer
            // 这个BufferedChannel其实就是个带WriteBuffer缓存的FileChannel
            // 写入这个BufferedChannel的数据有两部分组成：
            // 1. 写入到FileChannel的，已经flush到磁盘的
            // 2. 写入到这个WriteBuffer的
            // 其中这个writeBufferStartPosition表示文件中已经写入的位置，也表示这个writeBuffer开始的位置相当于
            // FileChannel中的哪个位置
            //
            // 下面这个if表示要读取的开始位置pos是在writeBuffer中
            if (writeBuffer != null && writeBufferStartPosition.get() <= pos) {
                int positionInBuffer = (int) (pos - writeBufferStartPosition.get());
                int bytesToCopy = Math.min(writeBuffer.writerIndex() - positionInBuffer, dest.writableBytes());

                if (bytesToCopy == 0) {
                    throw new IOException("Read past EOF");
                }

                dest.writeBytes(writeBuffer, positionInBuffer, bytesToCopy);
                pos += bytesToCopy;
                length -= bytesToCopy;
            } else if (writeBuffer == null && writeBufferStartPosition.get() <= pos) {
                // here we reach the end
                break;
                // first check if there is anything we can grab from the readBuffer
            } else if (readBufferStartPosition <= pos && pos < readBufferStartPosition + readBuffer.writerIndex()) {
                int positionInBuffer = (int) (pos - readBufferStartPosition);
                int bytesToCopy = Math.min(readBuffer.writerIndex() - positionInBuffer, dest.writableBytes());
                dest.writeBytes(readBuffer, positionInBuffer, bytesToCopy);
                pos += bytesToCopy;
                length -= bytesToCopy;
                // let's read it
            } else {
                readBufferStartPosition = pos;

                int readBytes = fileChannel.read(readBuffer.internalNioBuffer(0, readCapacity),
                        readBufferStartPosition);
                if (readBytes <= 0) {
                    throw new IOException("Reading from filechannel returned a non-positive value. Short read.");
                }
                readBuffer.writerIndex(readBytes);
            }
        }
        return (int) (pos - prevPos);
    }
```

