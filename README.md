# Chunked HTTP with Ajax and Scala

*April 16, 2014*

In Web programming, it is often handy to be able to process an HTTP request as 
a stream while sending back periodic status updates to the client.

Using [Java servlets](http://en.wikipedia.org/wiki/Java_Servlet), we can write 
a simple server that reads a request body line by line, and echos back to the 
client each line that it reads, along with the current progress:

```scala
override def doPost(req: HttpServletRequest, res: HttpServletResponse) {

  def progress(pos: Int) =
    req.getContentLength match {
      case x if x >= 0 =>
        (pos.toDouble / x.toDouble * 100).toInt.toString + "%"
      case _ =>
        "(progress unknown)"
    }

  val s = new Scanner(req.getInputStream)
  s.useDelimiter("[\r\n]+");

  while (s.hasNext) { // repeat until the request is completely processed

    val line = s.next // read a chunk from the request
      
    // simulate slow server-side processing of the chunk
    Thread.sleep(1000)

    // write the progress and the chunk to the response
    res.getWriter.write(progress(s.`match`.end) + ": " + line + "\n")

    // force the response buffer to be written immediately to the client
    res.flushBuffer

  }

}
```

We can test this using curl, observing each line returing about one second 
later than the last:

```
$ curl -N -X POST --data-binary @test.txt localhost:8080/echo
26%: Hello, world!
40%: Line 2
54%: Line 3
78%: One more...
98%: All done!
```

To interact with this service from JavaScript, we need an `XMLHttpRequest`:

```javascript
function XHR() {
  if (window.XMLHttpRequest) {
    // IE7+, Firefox, Chrome, Opera, Safari
    return new XMLHttpRequest();
  } else {
    // IE6, IE5
    return new ActiveXObject('Microsoft.XMLHTTP');
  }
}
```

We also need to be able to do something with the output:

```javascript
function println(html) {
  var div = document.createElement('div');
  div.innerHTML = html;
  document.getElementsByTagName('body')[0].appendChild(div);
}
```

Finally, we can make a request and handle the response.  We're watching for a 
`readyState` of `3`, which means [*processing request*](http://www.w3schools.com/ajax/ajax_xmlhttprequest_onreadystatechange.asp):

```javascript
function echo(xhr,data) {
  var len = 0;
  xhr.onreadystatechange =
    function() {
      if (xhr.readyState == 3 && xhr.status == 200) {
        var chunk = xhr.responseText.substr(len);
        len = len + chunk.length; 
        println(chunk);
      }
      if (xhr.readyState == 4 && xhr.status == 200) {
        println('<hr />');
      }
    }
  xhr.open('POST', '/echo', true);
  xhr.send(data);
}
```

Connecting this to a `textarea`, we get something like this:

![chunky-http screenshot](https://raw.githubusercontent.com/earldouglas/xwp-template/chunky/readme/chunky-http.png)

Check out the demo at [chunky-http.herokuapp.com](http://chunky-http.herokuapp.com/)
