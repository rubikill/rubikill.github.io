---
layout: post
title:  "Concurrent Programming với Elixir"
date:   2018-05-09 20:59:51 +0700
comments: true
categories: tech
---

Mục đích bài viết này để
1. Giới thiệu về phương pháp lập trình bất đồng bộ với cách tiếp cận khác những ngôn ngữ như Java hoặc C#.
2. Những bước đầu tiên để viết 1 chương trình chạy bất đồng bộ bằng Elixir
3. Hi vọng sẽ lôi kéo được thêm người học Elixir :)

# 1. Concurrent Programming

Concurrent Programming hiểu đơn giản là việc lập trình cho phần mềm có thể chia nhỏ tác vụ và chạy đồng thời, độc lập với nhau. Elixir là ngôn ngữ lập trình được viết dựa trên nền tảng ngôn ngữ erlang - ngôn ngữ giải quyết bài toán concurrent trên rất tốt.

Nếu trong Java hay C# việc lập trình bất đồng bộ bạn sẽ sữ dụng những phương pháp như Locking, Thread pool, mutex hay là semaphore... thì ở Elixir sẽ cài đặt trên mô hình **Actor model (1)**.

Actor là một đơn vị chính của mô hình Actor model và đảm nhận mọi thao tác tính toán trong mô hình này. Các đặc điểm chính của actors bao gồm:
 - message passing - các actor trao đổi với nhau bằng cách gửi message vào mailbox của nhau. Bạn muốn bảo một actor làm gì cho bạn: gửi message cho nó.
 Bạn muốn giết một actor: gửi message cho nó. Bạn muốn truy cập thông tin một actor: gửi message cho nó rồi check mailbox.
 - never share memory - mỗi actor có một state riêng mà không có actor nào có thể truy cập hay thay đổi được.
 - there are many actors - Actor là thứ sống theo bầy đàn. Trong mô hình này, tất cả đều là actor hoặc không có một actor nào cả. Đồng thời actor được định danh (giống như bạn được cha mẹ bạn cấp cho cái tên), tui sẽ nói rõ hơn về cái này trong phần tiếp theo.
 - asynchronous - mọi message đều là bất đồng bộ, tức là lúc bạn bấm gửi và lúc nào nó tới là hai chuyện khác nhau.

Khi nhận được một message, actor sẽ phải băn khoăn với 3 lựa chọn:
 - Xử lý thông tin và update state của nó.
 - Tạo thêm các actor khác.
 - Gửi message cho một actor khác.

> Đoạn này là copy trên Blog Quần Cam vì ảnh giải thích khá dễ hiểu về mô hình này, đọc thêm [tại đây](https://quan-cam.com/posts/elixir-erlang-actors-model-va-concurrency)

Có 1 tính chất trong elixir đó chính là **Immutability** mình xin tạm dịch là tính bất biến. Hãy cùng xem đoạn code dưới đây:

```
array = [1, 2, 3]
square(array)
print(array)
```

Giả sử hàm `square` sẽ bình phương giá trị từng phần tử trong mảng `array`. Bạn mong đợi kết quả trả về là [1, 4, 9]. Nhưng trong elixir kết quả sẽ trả về là `[1, 2, 3]` trừ khi bạn gán lại kết quả cho biến array như code dưới đây

```
array = [ 1, 2, 3 ]
array = square(array)
print(array)
```

Tính chất này giúp cho việc lập trình bất đồng bộ đơn giản hơn rất nhiều vì bạn không cần phải lo giá trị của 1 biến bị thay đổi bởi 1 process khác. Đây là 1 tính chất khá thú vị ở Elixir. Nhưng cái gì cũng sẽ có 2 mặt tốt và chưa tốt.
Bạn có thể tìm đọc thêm ở (2).

# 2. Làm quen với Concurrent Programming trong elixir

Nói 1 chút về process trong elixir. Process trong elixir khác với process chạy trên máy tính của bạn, nó chạy bên trong máy ảo erlang VM được quản lý trong ứng dụng của bạn. Trung bình 1 node trên chạy trên máy ảo erlang có thể tạo được 134 triệu processes con. Con số khá ấn tượng nhỉ :)

Việc khởi tạo 1 process trong elixir cũng rất đơn giản. Cùng xem ví dụ dưới đây:

```elixir
# Đoạn code này định nghĩa 1 module trong elixir và code 1 hàm là greet in ra màn
# hình là "Hello"
defmodule SpawnBasic do
  def greet do
    IO.puts "Hello"
  end
end

# Đây là cách gọi hàm thông thường
SpawnBasic.greet
Hello
:ok

# Còn đây là cách để chạy hàm trên trong 1 process
spawn(SpawnBasic, :greet, [])
Hello
#PID<0.42.0>
```

Kết quả trả về của hàm `spawn` là 1 địa chỉ tới process vừa khởi tạo. Tiếp theo chúng ta sẽ đến phần giao tiếp giữa các process với nhau thế nào.

Trong elixir có 2 hàm là **send** dùng để gửi message tới 1 process và hàm **receive** để nhận 1 message từ 1 process khác gọi sang. Thay đổi 1 chút đoạn code trên.

```elixir
defmodule Spawn do
  def greet do
    # Hàm này sẽ giúp nhận đối số từ bên ngoài
    receive do
      # sender là process gọi tới hàm này và msg là tin nhắn được gửi sang
      {sender, msg} ->
        # Dùng hàm send để gửi lại cho sender kết quả (*)
        send sender, { :ok, "Hello, #{msg}" }
    end
  end
end

# Khởi tạo process
pid = spawn(Spawn, :greet, [])
# Gửi tin nhắn tới process đó
send pid, {self, "World!"}
# Lắng nghe kết quả trả về
receive do
  # Chổ này sẽ match với giá trị trả về của hàm greet bên trên (*)
  {:ok, message} ->
    IO.puts message
end

Output:
> Hello, World!
```

Đoạn code trên nếu ta tiếp tục gọi thêm 1 message nữa
```
...
send pid, {self, "Ahihi!"}
receive do
  {:ok, message} ->
    IO.puts message
end

Output:
> Hello, World!
...stop...
```

Chương trình bị dừng thay vì tiếp tục trả về message là `Hello, Ahihi!`. Do sau khi hàm greet nhận được message thì đã gọi gửi trở lại cho main process và exit khỏi chương trình.

> Giải thích chỗ này 1 chút đó là function trong elixir sẽ trả về kết quả ở dòng  cuối cùng của thân hàm. Trong đoạn code trên hàm greet sẽ trả về pid là kết quả trả về của lời gọi hàm send

**Vậy làm sao để nhận được kết quả mong muốn là `Hello, Ahihi!` ở lần send message thứ 2?**
> Chúng ta sẽ thay đổi hàm greet để trả về lời gọi hàm chính nó để khi send tiếp tục lắng nghe tin nhắn từ main process

```
def greet do
  receive do
    {sender, msg} ->
      send sender, { :ok, "Hello, #{msg}" }
      greet
  end
end

Output:
> Hello, World!
> Hello, Ahihi!
```

**Đoạn code này trong giống đệ quy phải không?**
> Đúng rồi, đây là đệ quy trong elixir

Nếu trong những ngôn ngữ khác như Java thì khi gọi đệ quy tức là bạn push 1 frame vào stack và nếu bị gọi với số lượng lớn nó sẽ bị out of memory. Vấn đề này được giải quyết bằng kĩ thuật `tail-call optimization` có thể hiểu là nếu như hàm trả về chính nó thì chương trình chỉ đơn giản là nhảy về điểm bắt đầu của chương trình.

Đây là 1 bài toán tìm số factorial thứ n được viết bằng đệ quy

```
def factorial(0), do: 1
def factorial(n), do: n * factorial(n-1)

Với n = 3 chương trình sẽ chạy như sau
3 * factorial(3-1)
3 * 2 * factorial(2-1)
3 * 2 * 1 * factorial(1-1)
3 * 2 * 1 * 1
6
```

Nhưng đây không phải là `tail-call optimization` bởi vì kết quả trả về của hàm thứ 2 không trả về chính nó mà trả về kết quả của 1 phép nhân. Thay đổi 1 chút như bên dưới

```
def factorial(n), do: _fact(n, 1)
defp _fact(0, acc), do: acc
defp _fact(n, acc), do: _fact(n-1, acc*n)

Với n = 3
_fact(3-2, 1*3) => _fact(2, 3)
_fact(2-1, 3*2) => _fact(1, 6)
_fact(1-1, 6*1) => _fact(0, 6)
6
```
Chương trình này sẽ dùng biến acc để tracking lại giá trị của số trước nó, và chương trình sẽ chạy lùi từ n về 0. Nhưng như đã nói, mọi thứ đều có 2 mặt tốt và chưa tốt. Để hiểu thêm bạn có thể tham khảo ở (3)

**Nhưng nếu mình muốn mọi thứ đồng bộ nhưng vẫn chạy thành nhiều process có được không?**
> Được, cùng xem ví dụ dưới đây

```
defmodule Demo1 do
  def say do
    receive do
      n ->
        IO.inspect(n)
    end
  end
end

nums = 1..10

Enum.each(nums, fn i ->
  pid = spawn(Demo1, :say, [])
  send(pid, i)
end)

defmodule Demo2 do
  def say(pid) do
    receive do
      n ->
        send(pid, n)
    end
  end
end

Enum.each(nums, fn i ->
  pid = spawn(Demo2, :say, [self])
  send(pid, i)

  receive do
    n ->
      IO.inspect(n)
  end
end)

Output:
# Demo1 cách này sẽ giúp bạn khởi tạo và chạy các process mà không quan tâm giá trị trả về có đúng thứ tự không
1
3
2
7
4
10
8
6
5
9

# Demo2 sẽ đảm bảo kết quả trả về đúng thứ tự
1
2
3
4
5
6
7
8
9
10
```

Trong elixir có 1 hàm giúp ta link được 2 process với nhau đó là hàm `spawn_link`

```
defmodule Demo do
  import :timer, only: [sleep: 1]

  def die_soon do
    sleep(500)
    exit(:dead)
  end

  def run do
    # Process.flag(:trap_exit, true)
    spawn_link(Demo, :die_soon, [])

    receive do
      msg ->
        IO.puts("Message: #{inspect(msg)}")
    after
      1000 ->
        IO.puts("I'm still alive")
    end
  end
end

Demo.run()

Output:
** (EXIT from #PID<0.73.0>) :dead
```

Chương trình trên bị dừng khi process con bị chết, nó sẽ ảnh hưởng tới thằng cha làm chết theo. Để process cha không chết ta bỏ comment chỗ `Process.flag(:trap_exit, true)` lúc này chương trình không bị dừng mà output ra là

```
Message: {:EXIT, #PID<0.78.0>, :dead}
```

# 3. Kết
Hi vọng các bạn có một góc nhìn khác, một hướng tiếp cận khác về lập trình đa luồng và quan trọng nhất là sẽ có bạn tìm hiểu thử về Elixir sau khi đọc xong bài viết này. Nếu bạn có bất cứ thắc mắc hay câu hỏi nào thì vui lòng comment vào bên dưới bài viết. Mình sẽ cố gắng trả lời các câu hỏi nếu trong phạm vi hiểu biết :)

Đọc thêm:
 - (1) [Hewitt-ActorModel.pdf](http://worrydream.com/refs/Hewitt-ActorModel.pdf)
 - (2) [If-immutable-objects-are-good-why-do-people-keep-creating-mutable-objects](https://softwareengineering.stackexchange.com/questions/151733/if-immutable-objects-are-good-why-do-people-keep-creating-mutable-objects)
 - (3) [Tail-call optimization](https://pragtob.wordpress.com/2016/06/16/tail-call-optimization-in-elixir-erlang-not-as-efficient-and-important-as-you-probably-think/)

Ref: Elixir book
