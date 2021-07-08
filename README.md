# create-automation-jpa-entity
`JPA` 데이터베이스 기반 엔티티 클래스 원터치로 자동 생성하기 !
---

<br />

`JPA`를 사용하다 보면 데이터베이스의 테이블 명세를 보면서 엔티티 클래스를 작성하는 일이 자주 생긴다.

심지어 필드가 수십 개 정도 되면 엔티티 클래스를 만드는 일 자체가 무지막지한 노가다가 돼버리기 십상이다.

`intelliJ`는 이 과정을 지원해준다.

필요한 것은 오로지 `groovy script`뿐이다.

---

## **1. IntelliJ database tool 연동**

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FAqJNy%2Fbtq6fPZ8qSt%2FmHDBkiB7mpb8LPSVeibqy0%2Fimg.png" />

자신이 사용하고 있는 `데이터베이스를 연동`해준다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbZ4jSd%2Fbtq6gLW9erj%2FxK7ONFdFs31AgKirsAQtt1%2Fimg.png" />

접속 정보를 모두 알맞게 입력한 후 `Test Connection`을 눌러준 후 통과하면 OK를 눌러준다.

---

## **2. Groovy script 적용**

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbBudZh%2Fbtq58Zipvjz%2Fskbi36EqtGhKy4J0BRhmjk%2Fimg.png" />

테이블을 우클릭하여 위의 순서대로 클릭해준다.

그러면 `Generate POJOs.groovy` 라는 파일이 열린다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2F7GSKy%2Fbtq6aPUaFWI%2FRmmtDNp6gVvNEVjDkmkUf0%2Fimg.png" />

기본적인 script가 작성돼있는데, 해당 script를 그대로 사용하면 굉장히 이상한 엔티티 클래스가 만들어지므로

이를 입맛에 맞게 커스터마이징 해야 한다.

필자가 커스터마이징한 script는 `MSSQL`에 맞추긴 했는데 _~(회사가 MSSQL을 쓴다 😥)~_

그래도 대부분의 DB에서도 사용할 수 있을 것이라 여겨진다.~_(아닐 수도 있다.)_~

혹시라도 잘 맞지 않는다면 입맛대로 튜닝해서 사용하도록 하시라.

_**기존의 script를 모두 제거하고 아래의 script를 통째로 붙여 넣고 저장한다(CTRL + S).**_

```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.model.ObjectKind
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil
import com.intellij.psi.codeStyle.NameUtil

import javax.swing.*

/**
 * @author shirohoo* @link https://github.com/shirohoo/create-automation-jpa-entity
 * @param pakageName , primaryKey
 *
 * <pre>
 *
 *     this script's default primary key strategy is @GeneratedValue(strategy = GenerationType.IDENTITY)
 *     and specialized in Microsoft SQL Server
 *     and finally implemented Serializable so recommend that create serial version UID
 *
 *     first. enter your project package name. for example:
 *     >  com.intelliJ.psi
 *
 *     second. enter primary key column name of target database table.
 *     this script is convert input to camel case. for example 1:
 *     >  table primary key column name = MEMBER_ID
 *     >  enter primary key = memberId
 *
 *     example 2:
 *     >  table primary key column name = ID
 *     >  enter primary key = id
 *
 * </pre>
 */

columnType = [
        (~/(?i)bigint/)            : "Long",
        (~/(?i)int/)               : "Integer",
        (~/(?i)bit/)               : "Boolean",
        (~/(?i)decimal/)           : "BigDecimal",
        (~/(?i)float|double|real/) : "Double",
        (~/(?i)datetime|timestamp/): "LocalDateTime",
        (~/(?i)time/)              : "LocalTime",
        (~/(?i)date/)              : "LocalDate",
        (~/(?i)nvarchar/)          : "nvarchar",
        (~/(?i)varchar/)           : "varchar",
        (~/(?i)char/)              : "String"
]

def input = {
    JFrame jframe = new JFrame()
    String answer = JOptionPane.showInputDialog(jframe, it)
    jframe.dispose()
    answer
}

packageName = input("Enter your package name")
primaryKey = input("Enter column name of primary key ")

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter {
        it instanceof DasTable && it.getKind() == ObjectKind.TABLE
    }.each {
        generate(it, dir)
    }
}

def generate(table, dir) {
    def tableName = table.getName()
    def className = convertFieldName(tableName, true)
    def fields = categorizeFields(table)
    new File(dir, className + ".java").withPrintWriter {
        out -> generate(out, tableName, className, fields)
    }
}

def generate(out, tableName, className, fields) {
    out.println "package $packageName;"
    out.println ""
    out.println "import javax.persistence.*;"
    out.println "import java.io.Serializable;"
    out.println ""
    out.println "@Entity"
    out.println "@ToString @Getter"
    out.println "@NoArgsConstructor(access = AccessLevel.PROTECTED)"
    out.println "@Table(name = \"$tableName\")"
    out.println "public class $className extends BaseEntity {"
    out.println ""
    fields.each() {
        if (it.annos != "") {
            out.println " ${it.annos}"
        }
        if (it.name == primaryKey) {
            out.println " @Id @GeneratedValue(strategy = GenerationType.IDENTITY)"
        }
        if (it.type == 'nvarchar') {
            out.println " @Nationalized"
            out.println " @Column(name = \"${it.colName}\")"
            out.println " private String ${it.name};"
        } else if (it.type == 'varchar') {
            out.println " @Column(name = \"${it.colName}\")"
            out.println " private String ${it.name};"
        } else {
            out.println " @Column(name = \"${it.colName}\")"
            out.println " private ${it.type} ${it.name};"
        }
        out.println ""
    }
    out.println "}"
}

def categorizeFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())
        def typeStr = columnType.find {
            p, t -> p.matcher(spec).find()
        }.value
        fields += [[
                           colName: col.getName(),
                           name   : convertFieldName(col.getName(), false),
                           type   : typeStr,
                           annos  : ""]]
    }
}

def convertFieldName(str, capitalize) {
    def s = NameUtil.splitNameIntoWords(str)
            .collect {
                Case.LOWER.apply(it).capitalize()
            }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```

---

## **3. 엔티티 클래스 생성**

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbpCRyp%2Fbtq6gdM6627%2FVWyEOE5m7qfbP5dF17MgWK%2Fimg.png" />

이제 엔티티 클래스를 생성하고자 하는 테이블을 우클릭하여 위의 순서대로 클릭해준다.

그럼 입력창이 두 번 뜨고, 생성된 엔티티 클래스를 어떤 위치에 저장할 것인지 물을 것이다.

_**처음은 자신의 프로젝트 패키지명을 입력해주고,**_

_**두 번째는 테이블의 기본키 컬럼을 camel case로 변환한 이름을 입력해준다.**_

일반적으로 데이터베이스의 컬럼명은 대문자나 소문자의 `snake case`를 이용하는 게 관례이므로,

오로지 이 경우만 완벽하게 고려하여 작성하였다.

만약 다른 방식으로 사용하고 있다면 script를 변경해야 할 수도 있다.

```
// 예제1
테이블명: MEMBER
기본키 컬럼명: MEMBER_ID
입력해야 할 값: memberId


// 예제2
테이블명: MEMBER
기본키 컬럼명: ID
입력해야 할 값: id

```

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcWSmIg%2Fbtq6b1zWc6P%2FTG7uyE5bfkrhLrYW3BvIik%2Fimg.png" />

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Ft0bLB%2Fbtq6b08QQ51%2Fh4BViHdBNQHteHE7AZMOyK%2Fimg.png" />

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcHoqeD%2Fbtq6hTN2qqR%2F5YOIIH0Gg6U2FDngKfcNkK%2Fimg.png" />

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbmgUYy%2Fbtq57GJOf3Z%2FCXwPWSwIwCzKNPxiCOpcM1%2Fimg.png" />

그러면 이처럼 엔티티 클래스가 생성된다.

이 파일을 열어보면 아래와 같은 형식으로 작성돼있음을 확인할 수 있을 것이다. 

```java
package io.sirohoo.groovy;

import javax.persistence.*;
import java.io.Serializable;

@Entity
@Getter @ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "request_log")
public class RequestLog implements Serializable {

 @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
 @Column(name = "id")
 private Long id

 @Column(name = "mod_date")
 private LocalDateTime modDate

 @Column(name = "reg_date")
 private LocalDateTime regDate

 @Column(name = "client_ip")
 private String clientIp

 @Column(name = "http_method")
 private String httpMethod

 @Column(name = "request_uri")
 private String requestUri

}
```

---

## **4. serialVersionUID 생성**

`Hibernate docs`에서는 모든 엔티티 클래스에 대해 `Serializable` 인터페이스를 구현하는 걸 권장하고 있다.

대략 엔티티 매핑 방법에 따라 DB에 파라미터를 보낼 때 직렬화하여 보내야 하는 경우가 있기 때문이라고 설명하고 있다.

이 권고사항을 지키지 않고 기가막히게 상황이 맞아떨어질 경우 간혹 `Composite-id class must implement Serializable error` 같은걸 만날 수 있는데,

솔직히 아주 가끔 나오는 상황이라 굳이 해당 인터페이스를 구현하지 않아도 된다고 생각하긴 한다.

그래도 공식문서 권고사항이니 가급적 지키기 위해 `goorvy script`에 끼워넣어뒀다.

`intelliJ`는 직렬화시 필요한 `serialVersionUID`를 랜덤으로 생성해주는 기능이 있다.

먼저 Shift를 두 번 연속 빠르게 입력한다.

그러면 아래와 같은 창이 뜬다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FpbPNR%2Fbtq6eHVDJjR%2FrpsNHzFMvoosaAu6miDJJ1%2Fimg.png" />

```
Serializable class without 'se
```

위의 문자열을 붙여 넣어 검색하면 위의 기능이 검색되는데

저 기능을 ON으로 변경해주면 된다.

그리고 엔티티 클래스 이름에 마우스를 갖다 대면...

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fx2SOE%2Fbtq6cTPwzmw%2FBWknmMb3JpVEMkpKl1v3A1%2Fimg.png" />

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbPBn1r%2Fbtq6gLQpviF%2F7LQ88tJFqOP9AkO4wF9pN1%2Fimg.png" />
