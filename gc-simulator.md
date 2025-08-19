# IBM J9(SR4) Gencon GC 재현용 시스템 세팅 & 시뮬레이터 (Java)

> 목표: **AIX + IBM J9 SR4 + gencon** 환경에서 제공된 sample GC log와 **유사한 패턴**(큰 Nursery, 낮은 승격률, 8–13ms 수준 Scavenge pause, \~1.2GB/cycle 할당, 8–18초 간격)을 **WAS 탑재형 앱 + 배치 + 독립형 실시간 메트릭 수집/시각화**로 재현.
> 모든 상수는 **환경 변수**로 조정 가능하도록 설계.

---

## 목차

1. [전체 구조](#전체-구조)
2. [시스템 요구사항](#시스템-요구사항)
3. [JVM/GC 옵션 설정 (IBM J9 SR4)](#jvmgc-옵션-설정-ibm-j9-sr4)
4. [멀티모듈 프로젝트 구조](#멀티모듈-프로젝트-구조)
5. [코드](#코드)

   * [공용: Env 유틸](#공용-env-유틸)
   * [WAS 탑재용 앱(app-was)](#was-탑재용-앱app-was)
   * [배치 트랜잭션 발생기(txn-batch)](#배치-트랜잭션-발생기txn-batch)
   * [독립 메트릭 수집/시각화(metrics-agent)](#독립-메트릭-수집시각화metrics-agent)
6. [빌드 & 실행 매뉴얼](#빌드--실행-매뉴얼)
7. [재현 팁](#재현-팁)

---

## 전체 구조

* **app-was**: WAR 배포형 REST 애플리케이션

  * `POST /txn`: 1 TPS 온라인 트랜잭션.
  * 내부에 **에페메럴(단명) 객체 할당기**가 동작하여 **Nursery 중심 할당**을 유도(승격률 낮춤).
* **txn-batch**: 독립 실행형 Java 배치

  * 설정된 **시간 구간** 동안 **주기적으로 1 TPS**로 `/txn` 호출(스케줄링 포함).
* **metrics-agent**: 독립 실행형 메트릭 수집 & 대시보드

  * **JMX**(표준 + IBM MXBean)로 GC/메모리 지표 수집,
  * **SSE 기반 실시간 시각화**(내장 HTTP 서버 제공).

모든 구성요소는 **환경 변수**로 동작 파라미터를 주입.

---

## 시스템 요구사항

* **OS**: AIX 7.2 (ppc64) 권장
* **JVM**: IBM J9 SR4 (CompressedRefs 가능)
* **WAS**: WebSphere(Traditional 또는 Liberty) 중 택1
* **네트워크**: `metrics-agent` → `app-was` JMX 접근 가능 포트
* **빌드 도구**: Maven 3.8+ / JDK 8(IBM) 또는 11(IBM Semeru J9)

---

## JVM/GC 옵션 설정 (IBM J9 SR4)

> sample 로그 패턴에 맞춰 **Gencon + 고정힙 5GiB + 큰 Nursery + 64 GC 스레드**를 재현

**공통 JVM 옵션(서버 JVM, app-was에 적용)**

```bash
# AIX / IBM J9 SR4
-Xgcpolicy:gencon
-Xms5g -Xmx5g
# Nursery ~= 1.25 GiB (survivor ~128MiB 목표)
-Xmn1250m
# 대형 객체 영역(LOA) 유사화 (약 192MiB)
-Xloainitialsize:192m -Xloamaximumsize:192m
# GC 병렬 스레드
-Xgcthreads:64
# 페이지/압축참조(환경에 맞춰 자동)
-Xcompressedrefs
# verbosegc
-verbose:gc -Xverbosegclog:/var/log/was/verbosegc.%Y%m%d%H%M%S.xml
# JMX 원격(메트릭 수집을 위해)
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

> Liberty일 경우 `jvm.options` 파일, Traditional WAS는 서버 JVM 옵션 설정 콘솔/스크립트에서 동일하게 적용.
> LOA 고정은 환경에 따라 미세조정이 필요할 수 있습니다.

---

## 멀티모듈 프로젝트 구조

```
gc-repro/
├─ pom.xml
├─ common/                 # 공용 유틸 (환경변수 로딩 등)
│  └─ src/main/java/com/example/gc/common/Env.java
├─ app-was/                # WAR (WAS 탑재)
│  ├─ pom.xml
│  └─ src/main/java/com/example/gc/appwas/...
├─ txn-batch/              # 배치 실행형 Jar
│  ├─ pom.xml
│  └─ src/main/java/com/example/gc/txnbatch/TxnBatchMain.java
└─ metrics-agent/          # 메트릭 수집/시각화 실행형 Jar
   ├─ pom.xml
   └─ src/main/java/com/example/gc/metrics/...
```

**루트 `pom.xml` (핵심 부분)**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example.gc</groupId>
  <artifactId>gc-repro</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <modules>
    <module>common</module>
    <module>app-was</module>
    <module>txn-batch</module>
    <module>metrics-agent</module>
  </modules>

  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
</project>
```

---

## 코드

### 공용: Env 유틸

`common/src/main/java/com/example/gc/common/Env.java`

```java
package com.example.gc.common;

import java.time.Duration;

public final class Env {
    public static String str(String key, String def) {
        String v = System.getenv(key);
        return (v == null || v.isEmpty()) ? def : v;
    }
    public static int i(String key, int def) {
        String v = System.getenv(key);
        if (v == null || v.isEmpty()) return def;
        return Integer.parseInt(v);
    }
    public static long l(String key, long def) {
        String v = System.getenv(key);
        if (v == null || v.isEmpty()) return def;
        return Long.parseLong(v);
    }
    public static double d(String key, double def) {
        String v = System.getenv(key);
        if (v == null || v.isEmpty()) return def;
        return Double.parseDouble(v);
    }
    public static Duration durMillis(String key, long defMillis) {
        return Duration.ofMillis(l(key, defMillis));
    }
}
```

---

### WAS 탑재용 앱(app-was)

**기능**

* `POST /txn`: **온라인 트랜잭션** 1회 처리
* **에페메럴 할당기**: **단명 객체 중심** 할당으로 Nursery를 채우고 승격률 최소화
* 비즈니스 로직을 흉내 내는 **히프 churn** 제어(환경 변수)

**핵심 환경 변수**

* `APP_PORT` (기본 8080) — Liberty/TWAS는 서버 포트 사용
* `APP_ALLOC_MB_PER_SEC` (기본 90) — 초당 임시 힙 할당량
* `APP_TXN_HEAP_KB` (기본 256) — `/txn` 처리 중 일시 객체 총량
* `APP_SURVIVOR_RATE` (기본 0.01) — 살아남도록 보관할 비율(승격률 유도 방지: 낮게)
* `APP_BG_THREADS` (기본 4) — 백그라운드 할당 스레드 수
* `APP_CHUNK_KB` (기본 64) — 한 번에 할당할 청크 크기

**서블릿/리스너 예시**

`app-was/pom.xml` (WAR 생성, 서블릿 API)

```xml
<project ...>
  <parent>
    <groupId>com.example.gc</groupId><artifactId>gc-repro</artifactId><version>1.0.0</version>
  </parent>
  <artifactId>app-was</artifactId>
  <packaging>war</packaging>
  <dependencies>
    <dependency>
      <groupId>javax.servlet</groupId><artifactId>javax.servlet-api</artifactId><version>3.1.0</version><scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>com.example.gc</groupId><artifactId>common</artifactId><version>1.0.0</version>
    </dependency>
  </dependencies>
</project>
```

`AppContextListener.java` — 백그라운드 할당기

```java
package com.example.gc.appwas;

import com.example.gc.common.Env;
import javax.servlet.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

public class AppContextListener implements ServletContextListener {

    private ExecutorService pool;
    private final AtomicBoolean running = new AtomicBoolean(false);
    private final List<byte[]> survivors = Collections.synchronizedList(new ArrayList<byte[]>());

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        int bgThreads = Env.i("APP_BG_THREADS", 4);
        double mbPerSec = Env.d("APP_ALLOC_MB_PER_SEC", 90.0);
        int chunkKB = Env.i("APP_CHUNK_KB", 64);
        double survivorRate = Env.d("APP_SURVIVOR_RATE", 0.01);

        running.set(true);
        pool = Executors.newFixedThreadPool(bgThreads);

        for (int t = 0; t < bgThreads; t++) {
            pool.submit(() -> {
                final long chunk = chunkKB * 1024L;
                final long targetPerSec = (long) (mbPerSec * 1024 * 1024 / bgThreads);
                final Random rnd = new Random();
                long carried = 0L;
                while (running.get()) {
                    long start = System.nanoTime();
                    long allocated = 0L;
                    while (allocated < targetPerSec) {
                        byte[] b = new byte[(int) Math.min(chunk, 256 * 1024)]; // 작은 청크 위주
                        b[0] = 1; // 실제 사용
                        allocated += b.length;
                        // 극히 일부만 살아남게 해서 승격률 최소화
                        if (rnd.nextDouble() < survivorRate) {
                            survivors.add(b);
                            // 너무 쌓이지 않도록 상한 관리
                            if (survivors.size() > 5000) {
                                survivors.subList(0, 1000).clear();
                            }
                        }
                    }
                    long took = System.nanoTime() - start;
                    long sleepNs = 1_000_000_000L - took;
                    if (sleepNs > 0) LockSupport.parkNanos(sleepNs);
                }
            });
        }
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        running.set(false);
        if (pool != null) {
            pool.shutdownNow();
        }
    }
}
```

`TxnServlet.java` — 온라인 트랜잭션

```java
package com.example.gc.appwas;

import com.example.gc.common.Env;
import javax.servlet.http.*;
import javax.servlet.annotation.WebServlet;
import java.io.IOException;
import java.util.Random;

@WebServlet(name="TxnServlet", urlPatterns={"/txn"})
public class TxnServlet extends HttpServlet {
    private final Random rnd = new Random();

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // 트랜잭션 중 단명 객체 생성
        int txnHeapKB = Env.i("APP_TXN_HEAP_KB", 256);
        int chunk = Math.max(4 * 1024, Env.i("APP_CHUNK_KB", 64) * 1024); // 최소 4KB
        int toAlloc = txnHeapKB * 1024;
        int allocated = 0;
        byte[][] arr = new byte[Math.max(1, toAlloc / chunk)][];
        int idx = 0;

        while (allocated < toAlloc && idx < arr.length) {
            int size = Math.min(chunk, toAlloc - allocated);
            byte[] b = new byte[size];
            b[0] = 1;
            allocated += size;
            arr[idx++] = b;
        }

        // 약간의 CPU 혼합(짧은 계산)
        long sum = 0;
        for (int i = 0; i < 1000; i++) sum += rnd.nextInt(1000);
        resp.setStatus(200);
        resp.getWriter().write("{\"status\":\"OK\",\"sum\":" + sum + "}");
    }
}
```

`web.xml` (리스너 등록)

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="3.1">
  <listener>
    <listener-class>com.example.gc.appwas.AppContextListener</listener-class>
  </listener>
  <!-- /txn 서블릿은 @WebServlet으로 등록됨 -->
</web-app>
```

---

### 배치 트랜잭션 발생기(txn-batch)

**기능**

* 지정한 **시작/종료 시각** 사이에서 **주기(repeat interval)** 마다 \*\*지정 TPS(기본 1TPS)\*\*로 `/txn` 호출
* 내부 스케줄러 + 정밀 슬리핑으로 일정하게 보내기

**환경 변수**

* `BATCH_TARGET_URL` (예: `http://host:port/app/txn`)
* `BATCH_START` / `BATCH_END` (예: `2025-08-19T10:00:00`, 로컬 타임존)
* `BATCH_REPEAT_SEC` (기본 0; 0이면 단일 윈도만 실행)
* `BATCH_TPS` (기본 1)
* `BATCH_DURATION_SEC` (기본 900; 윈도 길이)
* `BATCH_CONN_TIMEOUT_MS` (기본 2000), `BATCH_READ_TIMEOUT_MS` (기본 5000)

`TxnBatchMain.java`

```java
package com.example.gc.txnbatch;

import com.example.gc.common.Env;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.concurrent.*;

public class TxnBatchMain {

    private static void runWindow(String target, int tps, int durationSec, int cto, int rto) throws Exception {
        long intervalNs = 1_000_000_000L / Math.max(1, tps);
        long end = System.nanoTime() + durationSec * 1_000_000_000L;
        while (System.nanoTime() < end) {
            long start = System.nanoTime();
            fireOnce(target, cto, rto);
            long took = System.nanoTime() - start;
            long sleep = intervalNs - took;
            if (sleep > 0) LockSupport.parkNanos(sleep);
        }
    }

    private static void fireOnce(String target, int cto, int rto) throws Exception {
        HttpURLConnection con = (HttpURLConnection) new URL(target).openConnection();
        con.setConnectTimeout(cto);
        con.setReadTimeout(rto);
        con.setRequestMethod("POST");
        con.setDoOutput(true);
        con.setRequestProperty("Content-Type", "application/json");
        try (OutputStream os = con.getOutputStream()) {
            os.write("{}".getBytes("UTF-8"));
        }
        int code = con.getResponseCode();
        if (code != 200) {
            System.err.println("Non-200: " + code);
        }
        con.disconnect();
    }

    public static void main(String[] args) throws Exception {
        String target = Env.str("BATCH_TARGET_URL", "http://127.0.0.1:9080/app/txn");
        int tps = Env.i("BATCH_TPS", 1);
        int duration = Env.i("BATCH_DURATION_SEC", 900);
        int repeatSec = Env.i("BATCH_REPEAT_SEC", 0);
        int cto = Env.i("BATCH_CONN_TIMEOUT_MS", 2000);
        int rto = Env.i("BATCH_READ_TIMEOUT_MS", 5000);

        DateTimeFormatter F = DateTimeFormatter.ISO_LOCAL_DATE_TIME;
        String s = Env.str("BATCH_START", "");
        String e = Env.str("BATCH_END", "");

        ZonedDateTime now = ZonedDateTime.now();
        ZonedDateTime start = s.isEmpty() ? now : LocalDateTime.parse(s, F).atZone(now.getZone());
        ZonedDateTime end = e.isEmpty() ? start.plusSeconds(duration) : LocalDateTime.parse(e, F).atZone(now.getZone());

        ScheduledExecutorService sch = Executors.newScheduledThreadPool(1);

        Runnable windowRun = () -> {
            try {
                System.out.println("Window start: " + ZonedDateTime.now());
                runWindow(target, tps, duration, cto, rto);
                System.out.println("Window done.");
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        };

        long delayToStartMs = Math.max(0, Duration.between(ZonedDateTime.now(), start).toMillis());
        sch.schedule(windowRun, delayToStartMs, TimeUnit.MILLISECONDS);

        if (repeatSec > 0) {
            sch.scheduleAtFixedRate(windowRun, delayToStartMs + repeatSec * 1000L, repeatSec * 1000L, TimeUnit.MILLISECONDS);
        }
    }
}
```

---

### 독립 메트릭 수집/시각화(metrics-agent)

**기능**

* JMX(MXBean)에서 **GC/메모리/스레드** 실시간 수집
* 내장 **HTTP 서버(포트 기본 9000)** + **SSE**로 프론트 차트 갱신
* IBM 전용 MXBean이 있으면 추가 지표(예: Scavenge/Global 구분, Nursery/Tenure free%)

**환경 변수**

* `METRICS_JMX_HOST` (기본 `127.0.0.1`)
* `METRICS_JMX_PORT` (기본 `9010`)
* `METRICS_HTTP_PORT` (기본 `9000`)
* `METRICS_POLL_MS` (기본 `1000`)

`metrics-agent/pom.xml` (내장 HTTP 서버로 JDK `com.sun.net.httpserver` 활용, 의존성 X)

```xml
<project ...>
  <parent>
    <groupId>com.example.gc</groupId><artifactId>gc-repro</artifactId><version>1.0.0</version>
  </parent>
  <artifactId>metrics-agent</artifactId>
  <dependencies>
    <dependency>
      <groupId>com.example.gc</groupId><artifactId>common</artifactId><version>1.0.0</version>
    </dependency>
  </dependencies>
</project>
```

`MetricsAgentMain.java`

```java
package com.example.gc.metrics;

import com.example.gc.common.Env;

import javax.management.MBeanServerConnection;
import javax.management.ObjectName;
import javax.management.remote.*;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.management.*;
import java.net.InetSocketAddress;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import com.sun.net.httpserver.HttpServer;

public class MetricsAgentMain {

    static class Snapshot {
        final long ts = System.currentTimeMillis();
        final long heapUsed, heapCommitted;
        final long edenUsed, survUsed, oldUsed;
        final long gcCount, gcTime;
        Snapshot(long heapUsed, long heapCommitted, long edenUsed, long survUsed, long oldUsed, long gcCount, long gcTime) {
            this.heapUsed = heapUsed; this.heapCommitted = heapCommitted;
            this.edenUsed = edenUsed; this.survUsed = survUsed; this.oldUsed = oldUsed;
            this.gcCount = gcCount; this.gcTime = gcTime;
        }
        String toJson() {
            return String.format(Locale.ROOT,
              "{\"ts\":%d,\"heapUsed\":%d,\"heapCommitted\":%d,\"edenUsed\":%d,\"survUsed\":%d,\"oldUsed\":%d,\"gcCount\":%d,\"gcTime\":%d}",
              ts, heapUsed, heapCommitted, edenUsed, survUsed, oldUsed, gcCount, gcTime);
        }
    }

    public static void main(String[] args) throws Exception {
        String host = Env.str("METRICS_JMX_HOST", "127.0.0.1");
        int port = Env.i("METRICS_JMX_PORT", 9010);
        int httpPort = Env.i("METRICS_HTTP_PORT", 9000);
        int pollMs = Env.i("METRICS_POLL_MS", 1000);

        JMXServiceURL url = new JMXServiceURL("service:jmx:rmi:///jndi/rmi://" + host + ":" + port + "/jmxrmi");
        JMXConnector jmxc = JMXConnectorFactory.connect(url, new HashMap<>());
        MBeanServerConnection mbsc = jmxc.getMBeanServerConnection();

        MemoryMXBean memBean = ManagementFactory.newPlatformMXBeanProxy(mbsc,
                ManagementFactory.MEMORY_MXBEAN_NAME, MemoryMXBean.class);
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();

        // 영역별(에덴/서바이버/올드) 추정: 이름에 따라 추론
        List<MemoryPoolMXBean> pools = ManagementFactory.getMemoryPoolMXBeans();
        MemoryPoolMXBean eden = findPool(pools, "eden|nursery");
        MemoryPoolMXBean surv = findPool(pools, "survivor|soa");
        MemoryPoolMXBean old  = findPool(pools, "tenure|old|soa");

        // HTTP 대시보드
        HttpServer server = HttpServer.create(new InetSocketAddress(httpPort), 0);
        List<OutputStream> sseClients = Collections.synchronizedList(new ArrayList<>());

        server.createContext("/", ex -> {
            byte[] html = DASHBOARD_HTML.getBytes("UTF-8");
            ex.getResponseHeaders().add("Content-Type", "text/html; charset=utf-8");
            ex.sendResponseHeaders(200, html.length);
            try (OutputStream os = ex.getResponseBody()) { os.write(html); }
        });

        server.createContext("/events", ex -> {
            ex.getResponseHeaders().add("Content-Type", "text/event-stream");
            ex.getResponseHeaders().add("Cache-Control", "no-cache");
            ex.sendResponseHeaders(200, 0);
            OutputStream os = ex.getResponseBody();
            synchronized (sseClients) { sseClients.add(os); }
        });

        server.start();
        System.out.println("Metrics dashboard: http://localhost:" + httpPort + "/");

        ScheduledExecutorService sch = Executors.newScheduledThreadPool(1);
        sch.scheduleAtFixedRate(() -> {
            try {
                long heapUsed = memBean.getHeapMemoryUsage().getUsed();
                long heapCommitted = memBean.getHeapMemoryUsage().getCommitted();
                long edenUsed = usage(eden);
                long survUsed = usage(surv);
                long oldUsed  = usage(old);

                long gcCount = 0, gcTime = 0;
                for (GarbageCollectorMXBean g : gcBeans) {
                    gcCount += g.getCollectionCount() < 0 ? 0 : g.getCollectionCount();
                    gcTime  += g.getCollectionTime() < 0 ? 0 : g.getCollectionTime();
                }
                Snapshot snap = new Snapshot(heapUsed, heapCommitted, edenUsed, survUsed, oldUsed, gcCount, gcTime);
                String s = "data:" + snap.toJson() + "\n\n";
                synchronized (sseClients) {
                    sseClients.removeIf(os -> !safeWrite(os, s));
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, 0, pollMs, TimeUnit.MILLISECONDS);
    }

    static MemoryPoolMXBean findPool(List<MemoryPoolMXBean> pools, String regex) {
        for (MemoryPoolMXBean p : pools) {
            String n = p.getName().toLowerCase(Locale.ROOT);
            if (n.matches(".*(" + regex + ").*")) return p;
        }
        return null;
    }
    static long usage(MemoryPoolMXBean p) {
        if (p == null || p.getUsage() == null) return -1;
        return p.getUsage().getUsed();
        // IBM J9 전용 세부(tenure/nursery %)는 com.ibm.lang.management.* MXBean으로 확장 가능
    }
    static boolean safeWrite(OutputStream os, String s) {
        try { os.write(s.getBytes("UTF-8")); os.flush(); return true; }
        catch (IOException e) { try { os.close(); } catch (IOException ignore) {} return false; }
    }

    static final String DASHBOARD_HTML =
        "<!doctype html><html><head><meta charset='utf-8'><title>GC Metrics</title></head>" +
        "<body><h2>GC Metrics (SSE)</h2>" +
        "<div>ts / heapUsed / edenUsed / survUsed / oldUsed / gcCount / gcTime(ms)</div>" +
        "<pre id='log' style='height:70vh;overflow:auto;background:#111;color:#0f0;padding:8px;'></pre>" +
        "<script>const log=document.getElementById('log');" +
        "const es=new EventSource('/events');es.onmessage=(e)=>{log.textContent+=e.data+'\\n';log.scrollTop=log.scrollHeight;};</script>" +
        "</body></html>";
}
```

---

## 빌드 & 실행 매뉴얼

### 1) 빌드

```bash
# IBM JDK/J9로 빌드 권장
mvn -q -DskipTests package
```

산출물:

* `app-was/target/app-was-1.0.0.war`
* `txn-batch/target/txn-batch-1.0.0-jar-with-dependencies.jar` (필요 시 shade)
* `metrics-agent/target/metrics-agent-1.0.0-jar-with-dependencies.jar`

> `txn-batch`/`metrics-agent`는 `maven-shade-plugin`으로 fat-jar 구성하는 것을 권장합니다.

### 2) WAS 배포

* **Liberty 예시**

  * `dropins/` 또는 `apps/`에 `app-was-1.0.0.war` 배치
  * `server.xml`에 애플리케이션 context root (예: `/app`) 설정
  * `jvm.options`에 [JVM/GC 옵션](#jvmgc-옵션-설정-ibm-j9-sr4) 추가
* **Traditional WAS**

  * Admin 콘솔 → Applications → Install new application → WAR 배포
  * 서버 JVM 옵션에 GC/JMX 설정 추가

**환경 변수 (app-was)**

```bash
export APP_BG_THREADS=4
export APP_ALLOC_MB_PER_SEC=90
export APP_TXN_HEAP_KB=256
export APP_SURVIVOR_RATE=0.01
export APP_CHUNK_KB=64
# 포트/컨텍스트는 WAS 설정에 따름
```

### 3) 메트릭 에이전트 기동

```bash
export METRICS_JMX_HOST=127.0.0.1
export METRICS_JMX_PORT=9010
export METRICS_HTTP_PORT=9000
export METRICS_POLL_MS=1000

java \
  -Xms256m -Xmx512m \
  -jar metrics-agent/target/metrics-agent-1.0.0-jar-with-dependencies.jar
# 브라우저에서 http://<host>:9000 접속
```

### 4) 배치 트랜잭션 발생기 실행

```bash
export BATCH_TARGET_URL="http://<was-host>:<port>/app/txn"
export BATCH_TPS=1
export BATCH_DURATION_SEC=1800         # 30분 윈도
export BATCH_REPEAT_SEC=0              # 반복 없음 (필요하면 초 단위 반복 주기)
export BATCH_START="2025-08-19T10:00:00"  # 생략 시 즉시 시작
export BATCH_END=""                    # 생략 시 START+duration

java -jar txn-batch/target/txn-batch-1.0.0-jar-with-dependencies.jar
```

* **주기적 실행(예: 매 정시 10분 동안)**: `BATCH_REPEAT_SEC=3600`, `BATCH_DURATION_SEC=600` 설정

---

## 재현 팁

1. **GC 패턴 맞춤(중요)**

   * `-Xms5g -Xmx5g -Xmn1250m -Xgcthreads:64`를 반드시 적용.
   * `APP_ALLOC_MB_PER_SEC`를 **70–120** 범위로 조정하며 **Scavenge 간격**을 **8–18초**대로 유도.
   * `APP_SURVIVOR_RATE`는 **0.005–0.02** 사이에서 미세 조정하면 \*\*승격률 최소화(tenure free \~47–53%)\*\*에 유리.
2. **verbosegc 확인**

   * `-Xverbosegclog:/var/log/was/verbosegc.%Y%m%d%H%M%S.xml` 로 로그 생성
   * Scavenge `durationms`가 **8–13ms**, 드물게 **\~25ms** 피크가 보이면 목표 달성.
3. **대형 객체 억제**

   * 코드에서 대형 배열(수 MB)을 만들지 말 것. `APP_CHUNK_KB`를 **64–128KB** 선으로 유지.

---

### 마무리

위 구성으로 **WAS 탑재형 온라인 트랜잭션(1 TPS)** + **백그라운드 단명 할당**을 결합해 **Nursery 중심 Scavenge**가 **주기적(8–18s)** 으로 발생하고 **pause가 수 ms\~수십 ms** 에 머무는 **sample GC log 유사 패턴**을 재현할 수 있습니다.
필요 시, `metrics-agent`에 **com.ibm.lang.management** MXBean을 추가로 연결해 **Nursery/Tenure 퍼센트**를 직접 노출하도록 확장 가능합니다.
