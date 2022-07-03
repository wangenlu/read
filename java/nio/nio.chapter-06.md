# Chapter 6 Buffers

NIO is based on buffers, whose contents are sent to or received from I/O
services via channels. This chapter introduces you to NIO’s buffer classes.

## Introducing Buffers

Buffers possess four properties:

- `Capacity`: The total number of data items that can be stored in the buffer. The capacity is specified when the buffer is created and cannot be changed later.
- `Limit`: The zero-based index of the first element that should not be read or written. In other words, it identifies the number of `live` data items in the buffer.
- `Position`: The zero-based index of the next data item that can be read or the location where the data item can be written.
- `Mark`: A zero-based index to which the buffer’s position will be reset when the buffer’s `reset()` method (presented shortly) is called. The mark is initially undefined.

## Buffer and its Children

`java.nio.Buffer`

## Buffer clear()

Clear this buffer. The position is set to 0, the limit is set to
the capacity, and the mark is discarded. This method
doesn’t erase the data in the buffer but is named as if it did
because it will most often be used in situations in which
that might as well be the case.

## Buffer flip()

Flip this buffer. The limit is set to the current position and
then the position is set to 0. When the mark is defined, it’s
discarded.

## boolean hasRemaining()

Return true when at least one element remains in this
buffer (that is, between the current position and the limit);
otherwise, return false.

## Buffer rewind()

Rewind and then return this buffer. The `position` is set to 0
and the `mark` is discarded.

## Buffer mark()

Set this buffer’s mark to its position and return this buffer.

## Buffer reset()

Reset this buffer’s position to the previously marked
position. Invoking this method neither changes nor
discards the mark’s value. This method throws
java.nio.InvalidMarkException when the mark hasn’t
been set; otherwise, it returns this buffer.

## Buffer Writing and Reading

`Tip` For maximum efficiency, you can perform bulk data transfers by using
the `ByteBuffer put(byte[] src)`, `ByteBuffer put(byte[] src,int offset, int length)`, `ByteBuffer get(byte[] dst)`, and
`ByteBuffer get(byte[] dst, int offset, int length)` methods to write and read an array of bytes.

## Flipping Buffers

> BufferDemo

```java
import java.nio.CharBuffer;

public class BufferDemo {
    public static void main(String[] args) {
        String[] poem = {"Roses are red", "Violets are blue", "Sugar is sweet", "And so are you."};
        CharBuffer buffer = CharBuffer.allocate(50);
        for (int i = 0; i < poem.length; i++) {
            // Fill the buffer.
            for (int j = 0; j < poem[i].length(); j++) {
                buffer.put(poem[i].charAt(j));
            }
            // Flip the buffer so that its contents can be read.
            buffer.flip();
            // Drain the buffer.
            while (buffer.hasRemaining()) {
                System.out.print(buffer.get());
            }
            // Empty the buffer to prevent BufferOverflowException.
            buffer.clear();
            System.out.println();
        }
    }
}
```
