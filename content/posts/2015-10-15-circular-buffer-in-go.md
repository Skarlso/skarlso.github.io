+++
author = "hannibal"
categories = ["Go", "problem solving", "programming"]
date = "2015-10-15"
title = "Circular buffer in Go"
url = "/2015/10/15/circular-buffer-in-go/"

+++

I'm proud of this one too. No peaking. I like how go let's you do this kind of stuff in a very nice way.

~~~go
package circular

import "fmt"

//TestVersion testVersion
const TestVersion = 1

//Buffer buffer type
type Buffer struct {
	buffer []byte
	full   int
	size   int
	s, e   int
}

//NewBuffer creates a new Buffer
func NewBuffer(size int) *Buffer {
	return &Buffer{buffer: make([]byte, size), s: 0, e: 0, size: size, full: 0}
}

//ReadByte reads a byte from b Buffer
func (b *Buffer) ReadByte() (byte, error) {
	if b.full == 0 {
		return 0, fmt.Errorf("Danger Will Robinson: %s", b)
	}
	readByte := b.buffer[b.s]
	b.s = (b.s + 1) % b.size
	b.full--
	return readByte, nil
}

//WriteByte writes c byte to the buffer
func (b *Buffer) WriteByte(c byte) error {
	if b.full+1 > b.size {
		return fmt.Errorf("Danger Will Robinson: %s", b)
	}
	b.buffer[b.e] = c
	b.e = (b.e + 1) % b.size
	b.full++
	return nil
}

//Overwrite overwrites the oldest byte in Buffer
func (b *Buffer) Overwrite(c byte) {
	b.buffer[b.s] = c
	b.s = (b.s + 1) % b.size
}

//Reset resets the buffer
func (b *Buffer) Reset() {
	*b = *NewBuffer(b.size)
}

func (b *Buffer) String() string {
	return fmt.Sprintf("Buffer: %d, %d, %d, %d", b.buffer, b.s, b.e, b.size)
}
~~~