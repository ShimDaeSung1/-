# 웹소켓 채팅 프로그램

* 사용 기술 :  JSP, 자바스크립트, 웹소켓

  * 채팅 서버 구현 ChatServer.java
![image](https://user-images.githubusercontent.com/86938974/166611318-d7450776-7a52-477c-a135-33dd5054b1b3.png)

```
package websocket;



import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

import javax.websocket.OnClose;
import javax.websocket.OnError;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;

@ServerEndpoint("/ChatingServer")
public class ChatServer {
	private static Set<Session> clients =
			Collections.synchronizedSet(new HashSet<Session>());
	
	@OnOpen //클라이언트 접속 시 실행
	public void onOpen(Session session) {
		clients.add(session); //세션 추가
		System.out.println("웹소켓 연결 : " + session.getId());
	}
	
	@OnMessage // 메시지를 받으면 실행
	public void onMessage(String message, Session session) throws IOException {
		System.out.println("메시지 전송 : "+session.getId()+":"+message);
		synchronized(clients) {
			for (Session client : clients) { //모든 클라이언트에 메시지 전달
				if(!client.equals(session)) {
					//단, 메시지를 보낸 클라이언트는 제외하고 전달
					client.getBasicRemote().sendText(message);
				}
				
			}
		}
	}
	
	@OnClose // 클라이언트와의 연결이 끊기면 실행
	public void onClose(Session session) {
		clients.remove(session);
		System.out.println("웹소켓 종료 : "+session.getId());
	}
	@OnError //에러 발생 시 실행
	public void onError(Throwable e) {
		System.out.println("에러 발생");
		e.printStackTrace();
	}

	
}

```
- @ServerEndpoint 애너테이션으로 웹소켓 서버의 요청명을 지정, 해당 요청명으로 접속하는 클라이언트를 이 클래스가 처리하게 한다.
- 웹소켓에 접속하기 위한 URL : (ws://호스트:포트번호/컨텍스트루트/ChatingServer)
- 새로 접속한 클라이언트의 세션을 저장할 컬렉션 생성, Collections 클래스의 synchronizedSet() 메서드는 멀티 스레드 환경에서도 안전한 Set 컬렉션을 생성해준다. => 여러 클라이언트가 동시 접속해도 문제가 생기지 않도록 동기화
- @OnOpen : 클라이언트가 접속했을 때 실행할 메서드, clients컬렉션에 세션 추가
- @OnMessage : 클라이언트로부터 메시지를 받았을 때 실행할 메서드, 클라이언트가 보낸 메시지와 연결된 세션이 매개변수로 넘어온다.
- @OnClose : 클라이언트가 접속 종료 시 실행할 메서드 정의
- @OnError : 에러 발생시 실행할 메서드

* 채팅 클라이언트 구현

  * 채팅 참여 화면 작성
![image](https://user-images.githubusercontent.com/86938974/166612136-2f619498-53fd-4c76-9365-cbece4ca6ab2.png)

```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>웹소켓 채팅</title>
</head>
<body>
	<script>
	function chatWinOpen(){
		var id = document.getElementById("chatId");
		if(id.value == ""){
			alert("대화명을 입력 후 채팅창을 열어주세요.");
			id.focus();
			return;
		}
		window.open("ChatWindow.jsp?chatId="+id.value,"","width=320, height=400");
		id.value = "";
	}
	</script>
	<h2>웹소켓 채팅 - 대화명 적용해서 채팅창 띄워주기</h2>
	대화명 : <input type="text" id ="chatId"/>
	<button onclick = "chatWinOpen();">채팅 참여</button>
</body>
</html>

```
- 채팅창을 팝업창으로 열어주는 함수 사용 chatWinOpen()
- 대화명 입력상자의 DOM객체를 얻어와 문제가 없다면 대화명을 매개변수로 전달해 채팅창을 띄운다.

  * 채팅창 작성
  - 채팅창으로 사용할 JSP 작성, 채팅서버에 접속하기 위한 요청명을 web.xml에 컨텍스트 초기화 매개변수로 저장

```
<context-param>
    <param-name>CHAT_ADDR</param-name>
    <param-value>ws://localhost:8081/MustHaveJSP</param-value>
  </context-param> 
</web-app>
```



```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
<script>
var webSocket = new WebSocket(
		"<%= application.getInitParameter("CHAT_ADDR") %>/ChatingServer");
var chatWindow, chatMessage, ChatId;

//채팅창이 열리면 대화창, 메시지 입력창, 대화명 표시란으로 사용할 DOM객체 저장
window.onload = function(){
	chatWindow = document.getElementById("chatWindow");
	chatMessage = document.getElementById("chatMessage");
	chatId = document.getElementById('chatId').value;
}

//메시지 전송
function sendMessage(){
	//대화창에 표시
	chatWindow.innerHTML += "<div class='myMsg'>"+chatMessage.value+"</div>"
	webSocket.send(chatId+'|' + chatMessage.value); //서버로 전송
	chatMessage.value = ""; // 메시지 입력창 내용 지우기
	chatWindow.scrollTop = chatWindow.scrollHeight; // 대화창 스크롤
}
//서버와의 연결 종료
function disconnect(){
	webSocket.close();
}
//엔터 키 입력 처리
function enterKey(){
	if(window.event.keyCode == 13){ //13은 'Enter' 키의 코드값
		sendMessage();	
	}
}


</script>
</head>
<body>

</body>
</html>
```
- 선언해둔 웹소켓 접속 URL뒤에 요청명을 붙여 웹소켓 객체를 생성하고 채팅창이 열리면 대화창, 메시지, 대화명으로 사용할 DOM객체를 변수에 저장한다.
- sendMessage()는 클라이언트의 메시지를 전송한다. enterKey()함수는 enter키가 눌리면 sendMessage()를 호출한다.

```
// 웹소켓 서버에 연결됐을 때 실행
webSocket.onopen = function(event){
	chatWindow.innerHTML += "웹소켓 서버에 연결되었습니다.<br/>";
};

//웹소켓이 닫혔을 때 <서버와의 연결이 끊겼을 때> 실행
webSocket.onclose = function(event){
	chatWindow.innerHTML += "웹소켓 서버가 종료되었습니다.<br/>";
};

//에러 발생 시 실행
webSocket.onerror = function(event){
	alert(event.data);
	chatWindow.innerHTML += "채팅 중 에러가 발생하였습니다. <br/>";
};

//메시지를 받았을 때 실행
webSocket.onmessage = function(event){
	var message = event.data.split("|"); // 대화명과 메시지 분리
	var sender = message[0]; //보낸 사람의 대화명
	var content = message[1]; //메시지 내용
	if(content != ""){
		if (content.match("/")){ //귓속말
			if(content.match(("/"+chatId))){ // 나에게 보낸 메시지만 출력
				var temp = content.replace("/" + chatId), "[귓속말] : ");
				chatWindow.innerHTML += "<div>"+sender + "" + temp + "</div>";
			}
		}
		else { //일반 대화
			chatWindow.innerHTML += "<div>" + sender + " : " + content + "</div>";
		}
	}
	chatWindow.scrollTOP = chatWindow.scrollHeight;
};
</script>
```
- 웹소켓 서버에 연결, 연결 종료, 에러발생, 메시지 받았을 때 각 상황은 이벤트로 전달되므로 이벤트별 리스너가 감지하여 메서드들을 호출한다. 호출시 인수로 이벤트 객체가 전달된다.
- 메시지 내용에 /가 포함되어 있다면 귓속말이다.

```
<style> 
#chatWindow{border:1px solid black; width:270px; height:310px; overflow:scroll; padding:5px;}
#chatMessage{width:236px; height:30px;}
#sendBtn{height:30px; postion:relative; top:2px; left:-2px;}
#closeBtn{margin-bottom:3px; position:relative; top:2px; left:-2px;}
#chatId{width:158px; height:24px; border:1px solid #AAAAAA; background-color:#EEEEEE;}
.myMsg{text-align:right;}	
</style>
</head>
<body>
	<!-- 대화창 UI 구조 정의 -->
	대화명 : <input type="text" id="chatId" value="${param.chatId}" readonly />
	<button id="closeBtn" onclick="disconnect();">채팅 종료</button>
	<div id="chatWindow"></div>
	<div>
		<input type = "text" id="chatMessage" onkeyup="enterKey();" >
		<button id = "sendBtn" onclick="sendMessage();">전송</button>
	</div>
```
* 동작 확인
  * MultiChatMain.jsp 실행

![image](https://user-images.githubusercontent.com/86938974/166614235-269e1620-0bf0-45cc-b31b-600b8241604f.png)

  * 총 3개의 채팅창 열기
    - 대화명 란에 "must"입력 -> 채팅 참여 클릭
    - 대화명 란에 "have"입력 -> 채팅 참여 클릭
    - 대화명 란에 "jsp"입력 -> 채팅 참여 클릭

![image](https://user-images.githubusercontent.com/86938974/166614588-ee23fdff-8b72-4bf6-8bfb-67f375e7cbe1.png)

  * 메시지 입력 후 전송 클릭, 또는 enter
![image](https://user-images.githubusercontent.com/86938974/166614613-a27ba3d5-d53d-4a3c-a4cb-b44040b58529.png)

  * 귓속말

![image](https://user-images.githubusercontent.com/86938974/166614671-899d3ce3-da3d-40c4-85b2-1d3941c693af.png)
![image](https://user-images.githubusercontent.com/86938974/166614690-de775797-34ed-4fb1-b1fb-ad8953edb0e5.png)


