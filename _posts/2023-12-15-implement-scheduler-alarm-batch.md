---
title: 스케쥴러를 사용한 푸시 알림과 배치 처리 기능 구현
author: jemlog
date: 2023-12-06 20:55:00 +0800
categories: [Project]
tags: [spring]
pin: false
image:
  path: '/assets/img/spring.webp'
---

이전 포스팅에서는 스프링에서 스케줄링을 사용하는 방법과 개선이 필요한 부분들을 살펴봤습니다. 먼저 마일스톤에는 어떤 기능들을 스케줄링으로 제공하는지 살펴본 뒤
앞에서 학습한 내용을 적용해나간 과정을 공유해보려 합니다.


## 푸시 알림 스케쥴러 개발
마일스톤에는 사용자가 선택한 **요일과 시간**에 리마인드 알림을 FCM 푸시를 통해 보내줍니다.

<img width="215" alt="스크린샷 2023-11-03 오후 2 18 13" src="https://github.com/dnd-side-project/dnd-9th-1-backend/assets/82302520/a88ccb88-2c91-4e4c-a4d1-faaa55aecd0a">

리마인드 알림 기능은 다음과 같은 규칙을 가지고 있습니다.
- 요일은 복수 선택이 가능하다.
- 시간 설정은 30분 단위로 가능하다. (ex. 7:00 -> 7:30 -> 8:00)
- 사용자는 알림을 받을지 여부를 직접 선택 가능하다.

### 구현 시 고려사항

**알림의 기준이 되는 목표 테이블과 요일을 저장하는 테이블**

<img width="471" alt="스크린샷 2023-11-08 오후 5 27 10" src="https://github.com/dnd-side-project/dnd-9th-1-backend/assets/82302520/0db20c50-455d-48b4-b3ae-23c8e7845db7">

**조회에 사용되는 컬럼**
- **alarm_enabled** : 알람을 보낼지 여부를 결정합니다.
- **alarm_time** : 알람을 보내는 시간을 결정합니다.
- **detail_goal_alarm_days.alarm_days** : 알람을 보내는 요일을 결정합니다.

### 코드 구현

**알림을 전송해야 하는 목표 리스트를 조회하는 쿼리**
```java

public List<DetailGoalAlarmResponse> getMemberIdListDetailGoalAlarmTimeArrived(DayOfWeek dayOfWeek, LocalTime alarmTime)
{
    return query.select(detailGoal)
                .from(detailGoal)
                .where(
                        ...
                        detailGoal.alarmEnabled.isTrue(), // 알람을 허용한 하위 댓글 조회
                        detailGoal.alarmDays.contains(dayOfWeek), // 알람을 보내기로한 요일들에 현재 요일이 포함되는지 체크
                        detailGoal.alarmTime.between(alarmTime.minusMinutes(1),alarmTime.plusMinutes(1)) // 미세한 시간차를 고려해서 앞뒤로 1분까지 범위에 포함
                )
                .fetch();
    }

}
```
**스케줄러 구현 코드**

```java
    @Scheduled(cron = "0 */30 * * * *", zone = "Asia/Seoul") // 30분 단위로 스케줄러가 동작합니다.
    public void sendAlarm()
    {
        DayOfWeek dayOfWeek = LocalDate.now().getDayOfWeek(); // 오늘이 어떤 요일인지 알아옵니다.
        LocalTime localTime = LocalTime.now(); // 현재 시간을 구합니다.
        LocalTime now = LocalTime.of(localTime.getHour(), localTime.getMinute(), 0);
         
        // 현재 요일과 시간에 해당하는 목표 리스트를 구해옵니다.
        List<DetailGoalAlarmResponse> detailGoalAlarmList = detailGoalQueryRepository.getMemberIdListDetailGoalAlarmTimeArrived(dayOfWeek, now);        

        // 조회한 목표의 사용자들에게 순차적으로 알림을 전송합니다.
        detailGoalAlarmList.forEach(alarmDto -> 
                        applicationEventPublisher.publishEvent(new AlarmEvent(alarmDto.uid(), alarmDto.detailGoalTitle())));
    }
```
DB에서 조회한 정보를 기반으로 유저들에게 FCM 푸시 알림을 전송합니다.

## 계획 상태 변경 배치 스케쥴러 개발

다음 기능은 달성 마감 기한이 지난 계획들을 실패 보관함으로 보내는 배치 처리 기능입니다. 해당 기능은 매일 00시에 일괄적으로 실행됩니다. 아래의 화면처럼 종료 날짜를 선택하면 나중에 보관함으로 들어가는 구조입니다.


<figure class="half">  
<a href="link">
<img width="200" alt="스크린샷 2023-12-16 오후 5 36 26" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/fb68c6c3-68f2-4b84-ac07-ea8d5f33313a">
</a> 
<a href="link">
<img width="200" alt="스크린샷 2023-12-16 오후 5 35 53" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/7ed4c89e-9449-4ef7-8683-7d81b28020f6">
</a>
</figure>

구현 코드는 다음과 같습니다.

```java
    @Scheduled(cron = "0 0 * * * *", zone = "Asia/Seoul")
    public void storeOutDateGoal() {

        // 현재 날짜에 아직 달성하지 못한 계획들을 조회
        List<Goal> goalList = goalQueryRepository.findGoalListEndDateExpired(LocalDate.now());
        // 보관 처리
        goalList.forEach(Goal::store);
    }
```

위의 방식으로 필요한 기능들은 쉽게 구현할 수 있었습니다. 하지만 발생 가능한 **예외 케이스**과 **분산 서버 환경에서의 운영**에 안전하게 대처하기 위해서는 몇가지 고민이 더 필요했습니다.

## 스케쥴러 기능 개선 사항

이전 포스팅에서 학습한 지식을 마일스톤에 적용 해보겠습니다.

### ThreadPoolTaskScheduler 적용
Spring의 `@Scheduler`를 적용하면 기본적으로 **모든 스케쥴러를 쓰레드 하나가 모두 관리합니다**. 하나의 스케줄러 쓰레드만으로 스케줄링을 처리한다면 어떤 일이 발생할 수 있을까요? 흔한 일은 아니지만 사용자가 계획 리마인드를 **자정**에 받도록 설정했다고 가정해보겠습니다. 자정에는 마감 기한을 지키지 못한 계획을 실패 보관함으로 보내는 배치 스케쥴링이 함께 돌아갑니다.
이때 배치 처리 스케쥴링이 먼저 스케줄러 쓰레드를 선점하면 **알림 전송 스케쥴러는 대기 상태에 들어갑니다**. 현재 사용할 수 있는 스레드가 없기 때문입니다.

```java
public List<DetailGoalAlarmResponse> getMemberIdListDetailGoalAlarmTimeArrived(DayOfWeek dayOfWeek, LocalTime currentTime)
{
    return query.select(detailGoal)
            .from(detailGoal)
            .where
                detailGoal.alarmTime.between(
                    currentTime.minusMinutes(2),
                    currentTime.plusMinutes(2) // 미세한 시간차를 고려해서 앞뒤로 2분까지 범위에 포함
                )
           .fetch();
    }

}
```
알림 대상을 판단하는 쿼리입니다. 만약 현재 스케쥴링을 진행하는 시각이 12시 30분이라면, 앞뒤로 2분 범위를 확인합니다. 범위로 조회를 하는 이유는 **예상치 못한 지연**으로 인해 현재 시각을 조회하는 메서드가 31분을 반환하면 정확한 쿼리가 불가능하기 때문입니다. 

만약 두개의 스케줄러 중 배치 스케줄러가 먼저 실행되고 작업 시간이 4~5분을 넘어간다면 어떻게 될까요? 배치 처리가 끝난 뒤 실행되는 알림 스케쥴러는 시간 차이로 인해 알림을 전송해야 하는 계획을 조회하지 못할 것입니다.

알림 리마인더를 현재는 30분 단위로 설정 가능하지만, 서비스가 성장하고 짧은 주기의 알림에 대한 사용자 니즈가 많으면 개선할 계획이 있습니다(실제로 1차 배포 후 알림 시간 다변화에 대한 요청이 있었습니다..!) 그때 단일 쓰레드로 인한 작업 지연 문제는 더 크게 다가올 것입니다.

이런 이유로 두 가지 스케쥴러를 실행하는 쓰레드를 분리하기로 결정했고, `ThreadPoolTaskScheduler`를 도입했습니다.

```java
@Configuration
@EnableScheduling
public class SchedulerConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();

        threadPoolTaskScheduler.setPoolSize(3); // 전체 쓰레드풀 사이즈
        threadPoolTaskScheduler.setThreadGroupName("scheduler thread pool");
        threadPoolTaskScheduler.setThreadNamePrefix("scheduler-thread-");
        threadPoolTaskScheduler.initialize();

        taskRegistrar.setTaskScheduler(threadPoolTaskScheduler);
    }
}
```
ThreadPoolTaskScheduler를 설정하는 Configuration 클래스입니다. 여기서 고민해봐야 하는 부분은 `poolSize` 설정입니다.

스케줄러는 **정해진 수의 작업**이 **일정한 시각에 동작**하기 때문에 새로운 스케줄러를 추가하지 않는 한 필요 이상의 쓰레드가 요구되지 않습니다. 사용량보다 쓰레드풀의 크기가 커지면
**유휴 쓰레드가 증가**하게 되고 이는 **메모리 낭비**와 **쓰레드간 컨텍스트 스위칭 비용 증가**로 이어집니다. 따라서 저는 `스케줄러 수 + 여유분 1`로 쓰레드풀 사이즈를 3으로 설정했습니다.  



### 비동기 처리의 필요성
이전 포스팅에서 스케줄러에 `@Async`를 사용해서 **쓰레드풀 기반 비동기 처리**를 하는걸 알아봤습니다. 스케줄링 서비스가 고도화될수록 꼭 필요한 기능이지만, 현재 마일스톤에 도입하는게 적절한지 몇 가지 검토를 진행했습니다. 새로운 기술을 적용할때 막연히 미래를 바라보고
이것 저것 추가하는건 개발자로서 꼭 피해야할 **오버 엔지니어링**이라 생각합니다.


비동기를 적용했을때 장점에 대해 생각해보겠습니다.
- 스케줄러의 **실행 시간이 길고 작업 간격이 짧을때** 비동기로 호출함으로써 작업 간의 지연 없이 독립적인 실행이 가능합니다.
- 스케줄러 쓰레드가 작업 수행 완료를 기다릴 필요가 없기에 ThreadPoolTaskScheduler를 효율적으로 사용할 수 있습니다.

현재 마일스톤은 매일 자정에 도는 스케쥴러와 30분 마다 도는 스케줄러가 있습니다. 스케쥴러가 실행되는 시간 간격이 꽤 넓은 편입니다. 또한 트래픽이 몰리는 API와 다르게 요청이 몰리지 않습니다. ThreadPoolTaskScheduler가 동기적으로 스케쥴링을 실행해도 문제가 없기에
비동기를 적용하는건 이득이 적다고 판단했습니다.

따라서 지금은 비동기 기능을 추가하지 않았습니다. 차후 **긴 작업 시간**과 **짧은 스케쥴링 간격**을 가진 스케쥴러를 도입한다면 비동기를 적용할 예정입니다. 

## 결론
스프링은 어노테이션 기반으로 간편하게 스케줄링을 사용하도록 지원해줍니다. 하지만 복잡도가 올라간 환경에서 사용할때는 추가적으로 고려해야할 사항이 많은 것 같습니다. 아마 로드밸런싱을 적용해서 분산 서버를 구축한다면 더 고민할 내용이 많아질 것이라 생각하는데요. 그때는 분산락을 사용한 동기화나 
별도의 알림 서버 구축을 고려해봐야 할 것 같습니다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
