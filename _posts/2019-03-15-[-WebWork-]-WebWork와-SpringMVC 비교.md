# SpringMVC�� WebWork ��  

����  
1. SpringMVC�� WebWork�� ������  
2. SpringMVC  
3. WebWork  
4. ���  

## SpringMVC�� WebWork �ֿ� ������  
### Controller vs Action  
- Controller  
    - �̱���  
    - Spring Bean ���� ���� �ȴ�  
        - Proxy ����� AOP ������ ����  
- Action  
    - �̱����� �ƴ� : �� ������Ʈ���� �ν��Ͻ� ����  

## SpringMVC  
> Spring�� �� ��ݿ��� ���� ���ϰ� �̿��ϱ� ���� �����ӿ�ũ  


![image.png](https://github.com/SeonheeKim/SeonheeKim.github.io/blob/master/content/images/SpringMVC.png?raw=true)


## WebWork  
> architecture of WebWork was based on the MVC Framework, Command, and Dispatcher patterns and the principle of IoC  
> 
> Apache Struts : Java EE �� ���ø����̼��� �����ϱ� ���� ���¼ҽ� �����ӿ�ũ.  
> Struts2�� ����ڿ��� ��û�� �ް� ��Ȱ�ϰ� �۵��� �� �ֵ��� ȯ���� �ٹ̰�,   
> � �׼��� ȣ���� �� ������ �� �׼��� �����Ѵ�. Result�� ���� ���� ���� �����͸� ����ڿ��� ��ȯ�ϴ� ���ҵ��� �����Ѵ�.  

### Lifecycle  
1. request begins when the servlet container receives a new request  
2. passed through **filter chain(set of filters) and sent to the FilterDispatcher**  
3. forwards the request to the **Actionmapper**  
4-1. if the request requires an action : sends an **Actionmapping object** back to FilterDispatcher  
4-2. not requires an action : ActionMapper returns **a null object**  
5. FilterDispatcher forwards the ActionMapper object to the **ActionProxy for further action.**  
6. ActionProxy invokes the Configuration File manager to get the attributes of the action. which is stored in the xwork.xml file and creates an **ActionInvocation object.**  
7. the ActionInvocation object contains attributes like the action, invocation context, result result code, etc.  
8. **Action result code to look up the appropriate result**, which is usually a JSP page.(I think is it the view)  
9. also ActionInvocation returns the response as a HttpServletResponse.  

![image.png](https://github.com/SeonheeKim/SeonheeKim.github.io/blob/master/content/images/WebWork.png?raw=true)

## ���  
 (I think?)  
 
| SpringMVC | WebWork |
| --- | --- |
| Controller | Action |
| Model | ActionInvocation Object |
| View Resolver | Action Mapper |
| View | Result |


<br>

���� : https://en.wikipedia.org/wiki/WebWork  
���� : http://theeye.pe.kr/archives/260  
���� : https://mer-bleu.tistory.com/24  
�̹��� ��ó : https://nhnent.dooray.com/project/2210798153185693714/2249870144271952624  
�̹��� ��ó : https://mer-bleu.tistory.com/24  