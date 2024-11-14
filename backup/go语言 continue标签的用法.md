```
package main
import "fmt"
func main() {
LABEL1:
	for i := 0; i <= 5; i++ {
		for j := 0; j <= 5; j++ {
			if j == 4 {
				continue LABEL1
			}
			fmt.Println(i, j)
		}
	}
}
```
//执行逻辑
// 当i=0时，内层循环j从0到5逐一递增。
// 当j=4时，if j == 4的条件为true，因此执行continue LABEL1。
// continue LABEL1会直接跳转到LABEL1标签所在的外层循环的下一个迭代，即i增加1，变为i=1。
// 这也意味着在i=0的情况下，当j=4及之后的内层循环（j=4到j=5）都不会执行fmt.Println(i, j)。
// 对于后续的i=1到i=5，同样的逻辑会重复执行：当j达到4时，再次执行continue LABEL1，跳过内层循环的剩余部分并开始外层循环的下一个迭代。
// 输出
// 因为在每次j等于4时都会跳过当前的外层循环迭代，程序输出如下：

// Copy code
// 0 0
// 0 1
// 0 2
// 0 3
// 1 0
// 1 1
// 1 2
// 1 3
// 2 0
// 2 1
// 2 2
// 2 3
// 3 0
// 3 1
// 3 2
// 3 3
// 4 0
// 4 1
// 4 2
// 4 3
// 5 0
// 5 1
// 5 2
// 5 3
// 可以看到，当j=4时，continue LABEL1跳过了当前i的剩余输出。`