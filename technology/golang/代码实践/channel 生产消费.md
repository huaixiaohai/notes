应用场景：

​		假设从数据库读取数据，经过数据转换转换后，写到其他的数据库，为了读取和写入操作并发，可如下实现

注意：

​		防止channel写满，但是消费者协程退出，导致生产者协程卡死

```go lago l
package main

import (
	"context"
	"reflect"
	"sync"
)

// 数据导入接口
type IDataImport interface {
	Read(limit int, start int) (interface{}, error)
	Convert(record interface{}) (interface{}, error)
	Write(rRecord interface{}, wRecord interface{}) error
}

const (
	channelSize = 20
	step        = 20
)

// 数据导入工具
type DataImport struct {
	ch chan interface{}
	ctx    context.Context
	cancel context.CancelFunc
	fs          IDataImport
	readInBatch bool
}

func NewDataImport(fs IDataImport) *DataImport {
	this := &DataImport{}
	this.ch = make(chan interface{}, channelSize)
	this.ctx, this.cancel = context.WithCancel(context.Background())
	this.fs = fs
	return this
}

func (t *DataImport) Run() error {
	var err error

	wg := &sync.WaitGroup{}
	wg.Add(2)

	// read
	go func() {
		defer wg.Done()
		e := t.read()
		if e != nil {
			t.cancel()
			err = e
		}
		close(t.ch)
	}()

	//write
	go func() {
		defer wg.Done()
		e := t.write()
		if e != nil {
			t.cancel()
			err = e
			<-t.ch   // 这一步主要是因为，当write发生错误，退出时，防止read协程卡死
		}
	}()

	wg.Wait()

	if err != nil {
    panic(err)
	}
	return nil
}

func (t *DataImport) read() error {
	ok := true
	index := 0
	var rowsSlicePtr interface{}
	var err error

	for ; ok; index += step {
		select {
		case <-t.ctx.Done():
			return nil
		default:
		}

		// 查询
		rowsSlicePtr, err = t.fs.Read(step, index)
		if err != nil {
			return err
		}
		sliceValue := reflect.Indirect(reflect.ValueOf(rowsSlicePtr))
		if sliceValue.Kind() != reflect.Slice {
			return errno.Unknown.CopyWithMessage("A point to a slice is needed")
		}
		if t.readInBatch {
			select {
			case <-t.ctx.Done():
				return nil
			default:
			}
			t.ch <- sliceValue.Interface()
			ok = sliceValue.Len() >= step
		} else {
			size := sliceValue.Len()
			for i := 0; i < size; i++ {
				select {
				case <-t.ctx.Done():
					return nil
				default:
				}
				t.ch <- sliceValue.Index(i).Interface()
			}
			ok = size >= step
		}
	}
	return nil
}

func (t *DataImport) write() error {
	var rRecord interface{}
	var wRecord interface{}
	var err error
	ok := true
	for ok {
		select {
		case <-t.ctx.Done():
		default:
		}

		rRecord, ok = <-t.ch
		if ok {
			// convert 也可以单独在run方法里面启动一个协程运行
			wRecord, err = t.fs.Convert(rRecord)
			if err != nil {
				return err
			}
			err = t.fs.Write(rRecord, wRecord)
			if err != nil {
				return err
			}
		}
	}
	return nil
}


// 实际应用，实现 IDataImport 接口
type MysqlDataImport struct {
  *DataImport
}

func NewMysqlDataImport() *MysqlDataImport {
  this := &MysqlDataImport{}
	this.DataImport = NewDataImport(this)
	return this
}

func Read(limit int, start int) (interface{}, error) {
  //TODO 做读取逻辑
}
func Convert(record interface{}) (interface{}, error) {
  //TODO 做转换逻辑
}
func Write(rRecord interface{}, wRecord interface{}) error {
  //TODO 写逻辑
}

func main() {
	NewMysqlDataImport().Run()
}


```

