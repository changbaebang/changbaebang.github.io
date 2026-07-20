---
layout: post
title: "AI와 Google Sheets로 필요한 앱 만들기: 아이 투약 기록 도구 공개"
date: 2026-07-20 09:20:00 +0900
tags: [chatgpt, ai-coding, google-sheets, google-apps-script, 아이투약기록, 소아투약기록, 어린이투약, 아기투약기록, 아이약기록, 아이약관리, 아기약관리, 투약일지, 투약기록표, 투약체크, 투약관리표, 복약기록, 아이주사기록, 소아주사기록, 약병잔량관리, 약병교체, 보호자공유, 가족공유, 가정투약관리, 육아앱, 육아도구, medication-tracker, parenting, personal-tool]
---

> 이 글에서 공개하는 도구는 보호자가 **이미 의료진에게 안내받은 투약 계획을 기록**하기 위한 개인용 보조 도구다.
> 약의 종류, 용량, 투약 간격, 휴약 여부를 판단하지 않으며 처방전과 의료진·약사의 안내를 대신하지 않는다.

이 글에서 하고 싶은 이야기는 세 가지다.

1. 이제 작은 앱은 내 생활과 취향에 맞게 직접 만들 수 있다.
2. 새로운 플랫폼을 매번 도입하기보다, 이미 쓰는 플랫폼 위에 필요한 기능을 빠르게 얹을 수 있다.
3. 요구사항 정리부터 코드, 디버깅, 설치 안내까지 AI와 함께 진행할 수 있다.

그리고 말로만 끝내지 않으려고, 실제로 사용 중인 설정과 공개용 전체 스크립트까지 함께 남긴다.

아이에게 약을 투여하는 기간에는 사소해 보이는 확인이 계속 필요하다.

- 오늘 투약했는가, 쉬었는가
- 지금 쓰는 약병에는 얼마나 남았는가
- 다음 투약 전에 새 병을 준비해야 하는가
- 보호자가 바뀌어도 같은 기록을 보고 있는가

메모 앱, 캘린더, 할 일 앱을 조합할 수도 있다. 하지만 사용하는 플랫폼이 이미 너무 많았다. 또 하나의 서비스를 찾아 가입하고 가족에게 사용법을 설명하기보다, 이미 함께 쓰고 있는 Google Sheets에 필요한 기능만 얹는 편이 빨랐다.

그래서 Google Sheets와 Apps Script로 작은 투약 기록 도구를 만들었다. 모바일에서는 `투약` 또는 `skip`만 체크하고, 계산과 약병 잔량 표시는 스프레드시트가 맡는다.

## TL;DR

- 앱을 내 취향과 생활 방식에 맞춰 만들었다.
- 저장·화면·공유 기능을 새로 만들지 않고 Google Sheets를 활용했다.
- 요구사항 정리, Apps Script 작성, 오류 수정, 설치 안내를 AI와 함께 진행했다.
- 실제 사용 설정과 전체 Apps Script를 공개한다.
- **의료적 판단 앱이 아니라, 이미 정해진 투약 계획을 기록하는 도구다.**

## 새 플랫폼을 더하지 않고, 쓰던 플랫폼을 활용했다

예전 같으면 먼저 앱스토어를 검색했을 것이다. 투약 기록, 가족 공유, 반복 일정, 약병 재고를 각각 지원하는 서비스를 찾아 비교했을 것이다.

그런데 작은 생활 문제는 요구사항이 아주 구체적이다. 우리에게 필요했던 것은 거대한 건강 관리 플랫폼이 아니었다.

```text
날짜별로 투약 또는 skip을 기록한다
-> 실제 투약한 만큼 약병 잔량을 계산한다
-> 다음 병이 필요할 때 미리 알려준다
-> 보호자가 모바일에서 같은 표를 본다
```

필요한 것이 이 정도로 분명하면, 적당한 서비스를 계속 찾는 비용이 직접 만드는 비용보다 커질 때가 있다. 그렇다고 저장, 권한, 모바일 화면, 가족 공유까지 모두 새로 만들 필요는 없다. Google Sheets가 이미 그 기반을 제공하고, AI와 Apps Script로 부족한 동작만 채울 수 있다.

처음부터 완성된 것은 아니었다. 체크박스를 하나로 합쳤다가 다시 둘로 나눴고, 미래 날짜를 어떻게 다룰지 바꿨으며, 첫 약병이 새 병이 아니라 사용 중이던 병일 때의 계산도 추가했다. 모바일에서 `TRUE/FALSE`가 그대로 보이는 문제와 약병 교체 기록이 예상치인지 실제 기록인지 헷갈리는 부분도 여러 번 고쳤다.

결국 중요한 것은 코드를 빨리 생성하는 일이 아니라, 실제로 쓰면서 **어떤 정보가 판단에 필요하고 무엇이 오히려 혼란을 만드는지** 계속 줄여가는 일이었다.

## AI와 함께 필요한 부분만 개발했다

처음부터 완성된 명세를 작성한 것은 아니다. 실제로 써보며 불편한 점을 한 문장씩 AI에 전달했다.

```text
투약 여부뿐 아니라 현재 병에 남은 양도 보고 싶다
-> 투약과 skip을 동시에 선택하면 안 된다
-> 첫 병은 새 병이 아니라 쓰던 병일 수 있다
-> 다음 기록에 필요한 병을 첫 줄에서 바로 알려달라
-> 모바일에서 TRUE/FALSE가 보이지 않게 해달라
```

AI는 요구사항을 Apps Script와 스프레드시트 수식으로 바꾸고, 나는 만들어진 시트를 직접 사용하면서 다시 요구사항을 수정했다. 코드를 한 번 생성하고 끝낸 것이 아니라, 생활 속 피드백을 다시 코드로 돌려보내는 방식이었다.

여기서 AI가 모든 것을 대신한 것은 아니다. Google Sheets는 데이터 저장, 모바일 화면, 접근 권한과 가족 공유를 맡았다. AI는 그 플랫폼 위에서 우리에게 필요한 계산과 상호작용을 빠르게 구현했다. 내가 해야 할 일은 불편을 구체적으로 말하고, 실제 결과를 확인하고, 앱이 판단하면 안 되는 경계를 정하는 것이었다.

## 무엇을 기록하는가

스크립트를 설치하면 세 개의 시트가 생긴다.

### 1. 약병 check

매일 사용하는 화면이다.

| 날짜 | 요일 | 투약 체크 | skip 체크 | 현재 병 남은 양 |
|---|---|---:|---:|---:|
| 2026-07-20 | 월 | ✓ |  | 12.4 mL |
| 2026-07-21 | 화 |  | ✓ | 12.4 mL |
| 2026-07-22 | 수 |  |  |  |

`투약`을 체크하면 같은 행의 `skip`이 해제되고, `skip`을 체크하면 `투약`이 해제된다. 처리하지 않은 날짜는 둘 다 비어 있다.

첫 번째 행에는 다음 미처리 날짜와 약병 준비 상태가 요약된다. 다만 이 안내는 입력한 설정을 계산해 보여주는 편의 기능이다. 실제 투약 여부와 방법은 처방 내용을 다시 확인해야 한다.

### 2. 병 교체 정보

실제로 투약하면서 새 병을 사용한 날짜만 모아 보여준다.

| 실제 교체일 | 이전 병 | 이전 병에서 사용한 양 | 새 병 | 새 병에서 사용한 양 | 투약 후 잔량 |
|---|---:|---:|---:|---:|---:|

예상 교체일을 미리 쌓는 표가 아니라, 체크한 기록에서 실제 교체가 발생했을 때만 남는 표다.

### 3. 설정

다음 값을 입력한다.

- 의료진에게 안내받은 1회 주입량
- 새 약병 한 병의 용량
- 처음 기록할 때 사용 중인 병의 실제 잔량
- 기록 시작일과 기록 일수
- 측정 가능한 최소 사용 단위

공개용 v8.1은 주입량, 약병 용량, 첫 병 잔량, 최소 사용 단위를 비워둔다. **처방전과 제품 안내에서 확인한 값을 사용자가 직접 입력해야 계산이 시작된다.**

## 내가 실제로 사용하는 설정

공개용 스크립트의 설정은 비워뒀지만, 이 도구가 어떤 문제에서 출발했는지 알 수 있도록 내가 실제로 사용하는 값도 공개한다.

| 설정 | 실제 사용값 | 의미 |
|---|---:|---|
| 1회 주입량 | 5.2 mL | 의료진에게 안내받은 1회량 |
| 새 약병 용량 | 20.0 mL | 새 병 한 병의 전체 용량 |
| 첫 병 실제 잔량 | 8.6 mL | 기록을 시작할 때 쓰고 있던 병의 잔량 |
| 최소 사용 단위 | 0.2 mL | 잔량 계산에 사용하는 최소 단위 |
| 기록 규칙 | 최근 7일 중 최대 6회 | 내가 안내받아 확인용으로 옮긴 일정 |

이 표는 **설치 예제가 아니라 내 기록을 재현하기 위한 실제 설정**이다. 같은 약을 사용한다고 해도 아이와 처방에 따라 값이 다를 수 있으므로 그대로 복사하면 안 된다. 공개용 스크립트가 이 값을 기본으로 채우지 않는 이유도 여기에 있다.

## 이 도구가 하지 않는 일

투약 관련 도구는 기능보다 경계가 중요하다.

이 스프레드시트는 다음을 판단하지 않는다.

- 아이에게 어떤 약이 필요한지
- 한 번에 얼마나 투여해야 하는지
- 며칠 연속 투약하거나 언제 쉬어야 하는지
- 남은 약을 다른 병과 나누어 사용해도 되는지
- 놓친 투약을 언제, 어떻게 보충해야 하는지

스크립트에 표시되는 횟수와 잔량은 사용자가 입력한 값을 바탕으로 한 계산 결과다. 기록이 누락되거나 설정이 잘못되면 결과도 틀린다. 약을 투여하기 전에는 처방전, 약 봉투, 제품 라벨과 의료진의 안내를 다시 확인해야 한다.

아이의 이름, 병명, 처방전 사진처럼 꼭 필요하지 않은 정보는 시트에 넣지 않는 편이 좋다. Google Sheets 공유 설정도 `링크가 있는 모든 사용자`가 아니라 가족의 지정된 Google 계정만 접근할 수 있도록 `제한됨`으로 유지한다.

## 공개용으로 바꾼 부분

개인용 v8에는 내 용량과 처방에 맞춘 안내가 들어 있었다. 공개용 v8.1에서는 개인 기본값과 직접적인 휴약·분할 투약 지시를 제거하고, 현재 스프레드시트 안에서 입력값을 기록하고 확인하는 도구로 정리했다. `최근 7일 중 7회` 경고도 모든 약의 기준이 아니라 내가 안내받은 일정에만 해당한다.

## 설치 방법

설치 전에는 기존 기록과 분리된 새 Google 스프레드시트에서 먼저 시험하는 것을 권한다.

1. Google Drive에서 빈 Google 스프레드시트를 만든다.
2. 기존 기록이 있는 스프레드시트에 적용한다면 `파일 -> 사본 만들기`로 먼저 백업한다.
3. 상단 메뉴에서 `확장 프로그램 -> Apps Script`를 연다.
4. 편집기에 들어 있는 기본 코드를 모두 지운다.
5. 아래 `전체 Apps Script 펼쳐보기`를 열어 코드를 모두 복사하고 붙여넣는다.
6. 저장한 뒤 함수 목록에서 `setupMedicationSheetV8`을 선택한다.
7. `실행`을 누르고 현재 스프레드시트를 수정하는 권한을 승인한다.
8. 스프레드시트로 돌아와 `설정` 시트의 값을 입력한다.
9. 테스트 날짜에 `투약`과 `skip`을 각각 체크해 반대쪽이 자동으로 해제되는지 확인한다.
10. 테스트가 끝난 뒤 실제 기록을 시작한다.

<details markdown="1">
<summary><strong>전체 Apps Script v8.1 펼쳐보기</strong></summary>

아래 코드를 전체 선택해 Apps Script 편집기에 붙여넣는다.

```javascript
// SPDX-License-Identifier: MIT
// Copyright (c) 2026 Changbae Bang

/**
 * @OnlyCurrentDoc
 */

/**
 * 약병 관리 v8.1 공개용
 *
 * 시트
 * - 설정
 * - 약병 check: 1행 실시간 요약 / 날짜 / 요일 / 투약 체크 / skip 체크 / 현재 병 남은 양
 * - 병 교체 정보: 실제 투약으로 새 병을 사용한 기록
 *
 * 운영 규칙
 * - 미래 날짜도 자유롭게 체크 가능
 * - 투약/skip 중 하나를 체크하면 반대쪽은 자동 해제
 * - 최근 7일 중 7회 입력은 삭제하지 않고 처방 확인 경고
 * - 남은 양이 1회 계산 주입량 이하이면 남은 양 셀을 강조
 * - 사용자가 입력한 최소 사용 단위로 내림 계산
 *
 * 주의
 * - 이 스크립트는 이미 의료진에게 안내받은 계획을 기록하는 보조 도구입니다.
 * - 용량, 간격, 휴약, 남은 약의 사용 방법을 결정하지 않습니다.
 * - 모든 설정값과 실제 투약 방법은 처방전, 제품 안내, 의료진·약사에게 확인하세요.
 */

const MED_V8 = Object.freeze({
  SETTINGS: '설정',
  CHECK: '약병 check',
  CHANGES: '병 교체 정보',
  FIRST_ROW: 3,
  MAX_DAYS: 730
});

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('약병 관리')
    .addItem('v8.1 시트 만들기 / 다시 설정', 'setupMedicationSheetV8')
    .addToUi();
}

function setupMedicationSheet() {
  setupMedicationSheetV8();
}

// 기존 함수명을 선택해 둔 경우를 위한 호환용
function setupMedicationSheetV6() {
  setupMedicationSheetV8();
}

// v7 함수명을 선택해 둔 경우를 위한 호환용
function setupMedicationSheetV7() {
  setupMedicationSheetV8();
}

function setupMedicationSheetV8() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const saved = readExistingData_(ss);

  const settings = resetSheet_(ss, MED_V8.SETTINGS, 20, 5);
  const check = resetSheet_(
    ss,
    MED_V8.CHECK,
    MED_V8.FIRST_ROW + MED_V8.MAX_DAYS - 1,
    23
  );
  const changes = resetSheet_(ss, MED_V8.CHANGES, MED_V8.MAX_DAYS + 10, 6);

  buildSettings_(settings, saved.settings);
  buildCheck_(check);
  restoreChecks_(check, settings, saved.actions);
  buildChanges_(changes);

  SpreadsheetApp.flush();
  ss.setActiveSheet(check);
  ss.toast('약병 관리 v8.1 설정이 완료되었습니다.', '약병 관리', 5);
}

/**
 * 모바일에서 투약과 skip이 동시에 체크되지 않도록 합니다.
 * 단순 onEdit 트리거라 별도 트리거 설치는 필요 없습니다.
 */
function onEdit(e) {
  if (!e || !e.range) return;

  const range = e.range;
  const sheet = range.getSheet();
  if (sheet.getName() !== MED_V8.CHECK) return;

  const firstRow = MED_V8.FIRST_ROW;
  const lastRow = firstRow + MED_V8.MAX_DAYS - 1;

  if (range.getLastRow() < firstRow || range.getRow() > lastRow) return;
  if (range.getLastColumn() < 3 || range.getColumn() > 4) return;

  // 일반적인 모바일 단일 체크
  if (range.getNumRows() === 1 && range.getNumColumns() === 1) {
    const row = range.getRow();
    const col = range.getColumn();
    const value = e.value == null ? '' : String(e.value);

    if (col === 3 && value === '투약') {
      sheet.getRange(row, 4).uncheck();
      return;
    }
    if (col === 4 && value === 'skip') {
      sheet.getRange(row, 3).uncheck();
      return;
    }
  }

  // 붙여넣기 등으로 두 체크가 동시에 들어온 경우 정리
  const startRow = Math.max(firstRow, range.getRow());
  const endRow = Math.min(lastRow, range.getLastRow());
  const rowCount = endRow - startRow + 1;
  const values = sheet.getRange(startRow, 3, rowCount, 2).getValues();
  const editedOnlySkip = range.getColumn() === 4 && range.getLastColumn() === 4;
  let changed = false;

  values.forEach(function(row) {
    if (row[0] === '투약' && row[1] === 'skip') {
      if (editedOnlySkip) row[0] = '';
      else row[1] = ''; // 나머지 경우 투약 우선
      changed = true;
    }
  });

  if (changed) {
    sheet.getRange(startRow, 3, rowCount, 2).setValues(values);
  }
}

function buildSettings_(sheet, saved) {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  const dose = numberOrBlank_(saved.dose);
  const capacity = numberOrBlank_(saved.capacity);
  const firstRemaining = numberOrBlank_(saved.firstRemaining);
  const startDate = dateOr_(saved.startDate, today);
  const days = integerBetween_(saved.days, 365, 1, MED_V8.MAX_DAYS);
  const unit = numberOrBlank_(saved.unit);

  sheet.getRange('A1:C1').merge();
  sheet.getRange('A1').setValue('설정');

  sheet.getRange('A2:C7').setValues([
    ['1회 주입량(ml)', dose, '처방에서 확인한 값을 직접 입력'],
    ['새 약병 용량(ml)', capacity, '두 번째 병부터 적용되는 새 병 전체 용량'],
    ['첫 병 실제 잔량(ml)', firstRemaining, '물려받은 병이면 현재 실제 잔량'],
    ['기록 시작일', startDate, '이 날짜부터 하루씩 생성'],
    ['기록 일수', days, '최대 ' + MED_V8.MAX_DAYS + '일'],
    ['최소 사용 단위(ml)', unit, '제품·측정 도구에서 확인한 값을 직접 입력']
  ]);

  sheet.getRange('A9').setValue('설정 상태');
  sheet.getRange('B9:C9').merge();

  // 내부 계산: 소수 오차를 줄인 뒤 최소 단위 개수로 내림
  sheet.getRange('D1:E5').setValues([
    ['내부 계산', '값'],
    ['주입 단위 수', ''],
    ['새 병 단위 수', ''],
    ['첫 병 단위 수', ''],
    ['설정 유효', '']
  ]);
  sheet.getRange('E2').setFormula('=IFERROR(ROUNDDOWN(ROUND(B2/B7,10),0),"")');
  sheet.getRange('E3').setFormula('=IFERROR(ROUNDDOWN(ROUND(B3/B7,10),0),"")');
  sheet.getRange('E4').setFormula('=IFERROR(ROUNDDOWN(ROUND(B4/B7,10),0),"")');
  sheet.getRange('E5').setFormula(
    '=AND(' +
      'B2<>"",B3<>"",B4<>"",B5<>"",' +
      'B6>=1,B6<=' + MED_V8.MAX_DAYS + ',B7>0,' +
      'E2>0,E3>0,E4>=0,E2<=E3,E4<=E3' +
    ')'
  );
  sheet.getRange('B9').setFormula(
    '=IF(E5=TRUE,' +
      '"설정 완료 · 계산 주입량 "&TEXT(E2*B7,"0.0")&"ml",' +
      '"새 약병 용량, 첫 병 잔량, 주입량을 확인하세요"' +
    ')'
  );

  const positive = SpreadsheetApp.newDataValidation()
    .requireNumberGreaterThan(0).setAllowInvalid(false).build();
  const nonNegative = SpreadsheetApp.newDataValidation()
    .requireNumberGreaterThanOrEqualTo(0).setAllowInvalid(false).build();
  const dayCount = SpreadsheetApp.newDataValidation()
    .requireNumberBetween(1, MED_V8.MAX_DAYS).setAllowInvalid(false).build();
  const dateValidation = SpreadsheetApp.newDataValidation()
    .requireDate().setAllowInvalid(false).build();

  sheet.getRange('B2:B3').setDataValidation(positive);
  sheet.getRange('B4').setDataValidation(nonNegative);
  sheet.getRange('B5').setDataValidation(dateValidation);
  sheet.getRange('B6').setDataValidation(dayCount);
  sheet.getRange('B7').setDataValidation(positive);

  styleTitle_(sheet.getRange('A1:C1'));
  sheet.setRowHeight(1, 34);
  sheet.getRange('A2:A7').setFontWeight('bold').setBackground('#D9EAF7');
  sheet.getRange('B2:B7').setBackground('#FFF2CC');
  sheet.getRange('A2:C7').setBorder(true, true, true, true, true, true);
  sheet.getRange('A9:C9').setBorder(true, true, true, true, true, true);
  sheet.getRange('A9').setFontWeight('bold').setBackground('#D9EAF7');
  sheet.getRange('B9').setWrap(true);
  sheet.getRange('B2:B4').setNumberFormat('0.0" ml"');
  sheet.getRange('B5').setNumberFormat('yyyy-mm-dd');
  sheet.getRange('B6').setNumberFormat('0');
  sheet.getRange('B7').setNumberFormat('0.0" ml"');
  sheet.setColumnWidth(1, 185);
  sheet.setColumnWidth(2, 145);
  sheet.setColumnWidth(3, 340);
  sheet.setFrozenRows(1);
  sheet.hideColumns(4, 2);
  sheet.setTabColor('#5B9BD5');

  sheet.getRange('B2').setNote('주입량이 최소 단위의 배수가 아니면 최소 단위로 내림 계산됩니다.');
  sheet.getRange('B4').setNote(
    '새 병으로 시작하면 새 약병 용량과 같은 값을 입력하세요. ' +
    '물려받은 병이면 현재 실제로 남은 사용 가능량을 입력하세요.'
  );
  sheet.getRange('B5').setNote('기록 후 시작일을 바꾸면 기존 체크가 다른 날짜로 이동할 수 있습니다.');
}

function buildCheck_(sheet) {
  const first = MED_V8.FIRST_ROW;
  const last = first + MED_V8.MAX_DAYS - 1;
  const S = quoteSheet_(MED_V8.SETTINGS);

  /*
   * 1행은 고정 제목이 아니라 다음 미처리 날짜의 실시간 요약입니다.
   * Q:W의 숨김 도우미 수식이 체크/skip 기록에 따라 요약을 계산합니다.
   */
  sheet.getRange('A1:E1').merge();
  sheet.getRange('A1').setFormula('=W1');

  sheet.getRange('A2:W2').setValues([[
    '날짜', '요일', '투약 체크', 'skip 체크', '현재 병 남은 양',
    '투약 전 병', '투약 전 잔량(단위)', '처리 상태',
    '현재 병 사용(단위)', '새 병 사용(단위)',
    '처리 후 병', '처리 후 잔량(단위)',
    '최근 7일 투약', '규칙 경고', '실제 교체', '잔량 경고',
    '다음 미처리 행', '다음 미처리 날짜', '직전 6일 투약',
    '다음 투약 전 병', '다음 투약 전 잔량(단위)',
    '요약 상태', '요약 문구'
  ]]);

  // 체크 시 사용자 값, 미체크 시 빈칸이므로 TRUE/FALSE가 노출되지 않습니다.
  sheet.getRange(first, 3, MED_V8.MAX_DAYS, 1).insertCheckboxes('투약');
  sheet.getRange(first, 4, MED_V8.MAX_DAYS, 1).insertCheckboxes('skip');

  const dateFormulas = [];
  const calcFormulas = [];

  for (let r = first; r <= last; r++) {
    const p = r - 1;
    const start7 = Math.max(first, r - 6);

    const date = `=IF(OR(${S}!$B$5="",ROW()-2>${S}!$B$6),"",${S}!$B$5+ROW()-3)`;
    const weekday = `=IF(A${r}="","",CHOOSE(WEEKDAY(A${r},1),"일","월","화","수","목","금","토"))`;
    dateFormulas.push([date, weekday]);

    const preBottle = r === first
      ? `=IF(OR(A${r}="",${S}!$E$5<>TRUE),"",1)`
      : `=IF(A${r}="","",K${p})`;

    const preRemaining = r === first
      ? `=IF(OR(A${r}="",${S}!$E$5<>TRUE),"",${S}!$E$4)`
      : `=IF(A${r}="","",L${p})`;

    const action =
      `=IF(A${r}="","",IF(AND(C${r}="투약",D${r}="skip"),"오류",` +
      `IF(C${r}="투약","투약",IF(D${r}="skip","skip",""))))`;

    const useCurrent =
      `=IF(OR(H${r}<>"투약",G${r}="",${S}!$E$5<>TRUE),0,MIN(G${r},${S}!$E$2))`;

    const useNew =
      `=IF(OR(H${r}<>"투약",G${r}="",${S}!$E$5<>TRUE),0,MAX(0,${S}!$E$2-G${r}))`;

    const postBottle =
      `=IF(OR(A${r}="",F${r}="",G${r}="",${S}!$E$5<>TRUE),"",` +
      `IF(AND(H${r}="투약",${S}!$E$2>G${r}),F${r}+1,F${r}))`;

    const postRemaining =
      `=IF(OR(A${r}="",G${r}="",${S}!$E$5<>TRUE),"",` +
      `IF(H${r}="투약",IF(${S}!$E$2<=G${r},G${r}-${S}!$E$2,` +
      `${S}!$E$3-(${S}!$E$2-G${r})),G${r}))`;

    const visibleRemaining =
      `=IF(OR(A${r}="",L${r}="",AND(H${r}<>"투약",H${r}<>"skip")),"",L${r}*${S}!$B$7)`;

    const count7 = `=IF(A${r}="","",COUNTIF(H${start7}:H${r},"투약"))`;
    const warning =
      `=IF(H${r}="오류","동시 체크",IF(AND(H${r}="투약",M${r}>=7),"7일 위반",""))`;
    const change =
      `=IF(AND(H${r}="투약",G${r}<>"",${S}!$E$5=TRUE,${S}!$E$2>G${r}),"교체","")`;
    const low =
      `=IF(AND(E${r}<>"",E${r}<=${S}!$E$2*${S}!$B$7),"잔량 주의","")`;

    calcFormulas.push([
      visibleRemaining, preBottle, preRemaining, action,
      useCurrent, useNew, postBottle, postRemaining,
      count7, warning, change, low
    ]);
  }

  sheet.getRange(first, 1, MED_V8.MAX_DAYS, 2).setFormulas(dateFormulas);
  sheet.getRange(first, 5, MED_V8.MAX_DAYS, 12).setFormulas(calcFormulas);

  /*
   * Q1: 첫 번째 미처리 또는 오류 행
   * R1: 해당 날짜
   * S1: 그 날짜 직전 6일의 실제 투약 횟수
   * T1/U1: 그 날짜의 투약 전 병 번호와 잔량
   * V1/W1: 요약 상태와 사용자용 문구
   */
  sheet.getRange('Q1').setFormula(
    `=IFERROR(MATCH(1,ARRAYFORMULA((A${first}:A${last}<>"")*` +
    `(H${first}:H${last}<>"투약")*(H${first}:H${last}<>"skip")),0)+${first - 1},"")`
  );
  sheet.getRange('R1').setFormula('=IF(Q1="","",INDEX(A:A,Q1))');
  sheet.getRange('S1').setFormula(
    `=IF(Q1="","",IF(Q1<=${first},0,` +
    `COUNTIF(INDIRECT("H"&MAX(${first},Q1-6)&":H"&(Q1-1)),"투약")))`
  );
  sheet.getRange('T1').setFormula('=IF(Q1="","",INDEX(F:F,Q1))');
  sheet.getRange('U1').setFormula('=IF(Q1="","",INDEX(G:G,Q1))');
  sheet.getRange('V1').setFormula(
    `=IF(${S}!$E$5<>TRUE,"설정",` +
    `IF(Q1="","완료",` +
    `IF(INDEX(H:H,Q1)="오류","오류",` +
    `IF(S1>=6,"규칙",` +
    `IF(U1<${S}!$E$2,"교체",` +
    `IF(U1<2*${S}!$E$2,"준비","정상"))))))`
  );

  const summaryFormula =
    `=IF(${S}!$E$5<>TRUE,` +
      `"⚙️ 설정 시트에서 주입량, 새 약병 용량, 첫 병 잔량을 확인하세요.",` +
    `IF(Q1="",` +
      `"✅ 생성된 날짜를 모두 처리했습니다.",` +
    `IF(INDEX(H:H,Q1)="오류",` +
      `"🔴 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 투약과 skip이 동시에 체크되었습니다."&CHAR(10)&` +
      `"둘 중 하나만 남겨 주세요.",` +
    `IF(S1>=6,` +
      `"🔴 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 이 날짜에 투약하면 최근 7일 중 7회가 됩니다."&CHAR(10)&` +
      `"설정한 일정과 다를 수 있습니다. 투약 전에 처방 또는 의료진 안내를 확인하세요.",` +
    `IF(U1>${S}!$E$2,` +
      `IF(U1-${S}!$E$2<${S}!$E$2,` +
        `"🟡 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 입력값 기준 병 #"&TEXT(T1,"0")&"의 예상 잔량입니다."&CHAR(10)&` +
        `"현재 "&TEXT(U1*${S}!$B$7,"0.0")&"ml · 기록한 1회량 "&TEXT(${S}!$E$2*${S}!$B$7,"0.0")&"ml · 처리 후 예상 "&TEXT((U1-${S}!$E$2)*${S}!$B$7,"0.0")&"ml · 다음 병을 준비하고 실제 사용법을 확인하세요.",` +
        `"🟢 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 입력값 기준 병 #"&TEXT(T1,"0")&"의 예상 잔량입니다."&CHAR(10)&` +
        `"현재 "&TEXT(U1*${S}!$B$7,"0.0")&"ml · 기록한 1회량 "&TEXT(${S}!$E$2*${S}!$B$7,"0.0")&"ml · 처리 후 예상 "&TEXT((U1-${S}!$E$2)*${S}!$B$7,"0.0")&"ml"` +
      `),` +
    `IF(U1=${S}!$E$2,` +
      `"🟡 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 입력값 기준 병 #"&TEXT(T1,"0")&"의 예상 잔량이 한 번분입니다."&CHAR(10)&` +
      `"다음 병 #"&TEXT(T1+1,"0")&"을 준비하고 실제 사용법을 처방 또는 제품 안내에서 확인하세요.",` +
    `IF(U1>0,` +
      `"🟠 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 현재 병 잔량이 입력한 1회량보다 적습니다."&CHAR(10)&` +
      `"새 병 #"&TEXT(T1+1,"0")&"을 준비하고 남은 약과 새 병의 사용 방법을 처방 또는 제품 안내에서 확인하세요.",` +
      `"🔴 "&TEXT(R1,"yyyy-mm-dd")&" ("&INDEX(B:B,Q1)&") · 현재 병 #"&TEXT(T1,"0")&"에 사용 가능한 약이 없습니다."&CHAR(10)&` +
      `"새 병 #"&TEXT(T1+1,"0")&"을 준비하고 실제 사용법을 처방 또는 제품 안내에서 확인하세요."` +
    `)))))))`;

  sheet.getRange('W1').setFormula(summaryFormula);

  // 1행 요약 영역 서식
  sheet.getRange('A1:E1')
    .setBackground('#D9EAF7')
    .setFontColor('#1F1F1F')
    .setFontWeight('bold')
    .setFontSize(11)
    .setHorizontalAlignment('left')
    .setVerticalAlignment('middle')
    .setWrap(true)
    .setBorder(true, true, true, true, false, false);
  sheet.setRowHeight(1, 72);

  styleHeader_(sheet.getRange('A2:W2'));
  sheet.setRowHeight(2, 32);
  sheet.getRange(first, 1, MED_V8.MAX_DAYS, 5)
    .setBorder(true, true, true, true, true, true)
    .setVerticalAlignment('middle');
  sheet.getRange(first, 1, MED_V8.MAX_DAYS, 4).setHorizontalAlignment('center');
  sheet.getRange(first, 5, MED_V8.MAX_DAYS, 1).setHorizontalAlignment('right');
  sheet.getRange(first, 1, MED_V8.MAX_DAYS, 1).setNumberFormat('yyyy-mm-dd');
  sheet.getRange(first, 5, MED_V8.MAX_DAYS, 1).setNumberFormat('0.0" ml"');
  sheet.setColumnWidth(1, 115);
  sheet.setColumnWidth(2, 55);
  sheet.setColumnWidth(3, 95);
  sheet.setColumnWidth(4, 95);
  sheet.setColumnWidth(5, 145);
  sheet.setFrozenRows(2);
  sheet.hideColumns(6, 18); // F:W 내부 수식 및 요약 도우미
  sheet.setTabColor('#70AD47');

  sheet.getRange('A1').setNote(
    '오늘 날짜가 아니라 날짜 순서상 첫 번째 미처리 행을 기준으로 안내합니다. ' +
    '투약 또는 skip을 선택하면 다음 미처리 날짜로 자동 이동합니다.'
  );
  sheet.getRange('C2').setNote(
    '투약했다면 체크합니다. skip은 자동 해제됩니다. ' +
    '최근 7일 중 7회 투약이어도 기록은 유지되고 행만 빨간색으로 표시됩니다.'
  );
  sheet.getRange('D2').setNote('투약하지 않은 날에 체크합니다. 투약 체크는 자동 해제됩니다.');
  sheet.getRange('E2').setNote(
    '선택 처리 후 현재 사용 중인 병의 남은 양입니다. 1회 계산 주입량 이하이면 색이 바뀝니다.'
  );

  const summary = sheet.getRange('A1:E1');
  const rows = sheet.getRange(first, 1, MED_V8.MAX_DAYS, 5);
  const remaining = sheet.getRange(first, 5, MED_V8.MAX_DAYS, 1);

  sheet.setConditionalFormatRules([
    // 동적 요약 색상
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="규칙"')
      .setBackground('#F4CCCC').setFontColor('#9C0006').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="오류"')
      .setBackground('#EADCF8').setFontColor('#6A1B9A').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="교체"')
      .setBackground('#FCE5CD').setFontColor('#B45F06').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="준비"')
      .setBackground('#FFF2CC').setFontColor('#7F6000').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="정상"')
      .setBackground('#E2F0D9').setFontColor('#274E13').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="완료"')
      .setBackground('#D9EAF7').setFontColor('#1F4E78').setBold(true)
      .setRanges([summary]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$V$1="설정"')
      .setBackground('#F2F2F2').setFontColor('#444444').setBold(true)
      .setRanges([summary]).build(),

    // 행 및 잔량 색상
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$P3="잔량 주의"')
      .setBackground('#FCE8B2').setFontColor('#B45F06').setBold(true)
      .setRanges([remaining]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$N3="동시 체크"')
      .setBackground('#EADCF8').setFontColor('#6A1B9A').setBold(true)
      .setRanges([rows]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$N3="7일 위반"')
      .setBackground('#F4CCCC').setFontColor('#9C0006').setBold(true)
      .setRanges([rows]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=AND($O3="교체",$N3="")')
      .setBackground('#DDEBF7').setBold(true).setRanges([rows]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=AND($H3="투약",$O3<>"교체",$N3="")')
      .setBackground('#E2F0D9').setRanges([rows]).build(),
    SpreadsheetApp.newConditionalFormatRule()
      .whenFormulaSatisfied('=$H3="skip"')
      .setBackground('#F2F2F2').setRanges([rows]).build()
  ]);
}

function buildChanges_(sheet) {
  const first = MED_V8.FIRST_ROW;
  const last = first + MED_V8.MAX_DAYS - 1;
  const C = quoteSheet_(MED_V8.CHECK);
  const S = quoteSheet_(MED_V8.SETTINGS);
  const condition = `${C}!$O$${first}:$O$${last}="교체"`;

  sheet.getRange('A1:F1').merge();
  sheet.getRange('A1').setValue('병 교체 정보');
  sheet.getRange('A2:F2').setValues([[
    '실제 교체일', '이전 병', '이전 병에서 넣은 양',
    '새 병', '새 병에서 넣은 양', '투약 후 새 병 잔량'
  ]]);

  sheet.getRange('A3').setFormula(`=IFERROR(FILTER(${C}!$A$${first}:$A$${last},${condition}),"")`);
  sheet.getRange('B3').setFormula(`=IFERROR(FILTER(${C}!$F$${first}:$F$${last},${condition}),"")`);
  sheet.getRange('C3').setFormula(`=IFERROR(FILTER(${C}!$I$${first}:$I$${last},${condition})*${S}!$B$7,"")`);
  sheet.getRange('D3').setFormula(`=IFERROR(FILTER(${C}!$K$${first}:$K$${last},${condition}),"")`);
  sheet.getRange('E3').setFormula(`=IFERROR(FILTER(${C}!$J$${first}:$J$${last},${condition})*${S}!$B$7,"")`);
  sheet.getRange('F3').setFormula(`=IFERROR(FILTER(${C}!$L$${first}:$L$${last},${condition})*${S}!$B$7,"")`);

  styleTitle_(sheet.getRange('A1:F1'));
  sheet.setRowHeight(1, 34);
  styleHeader_(sheet.getRange('A2:F2'));
  sheet.getRange('A2:F2').setWrap(true);
  sheet.setRowHeight(2, 42);
  sheet.getRange(3, 1, MED_V8.MAX_DAYS, 6)
    .setBorder(true, true, true, true, true, true)
    .setVerticalAlignment('middle');
  sheet.getRange(3, 1, MED_V8.MAX_DAYS, 1).setNumberFormat('yyyy-mm-dd');
  sheet.getRange(3, 2, MED_V8.MAX_DAYS, 1).setNumberFormat('"병 #"0');
  sheet.getRange(3, 4, MED_V8.MAX_DAYS, 1).setNumberFormat('"병 #"0');
  sheet.getRange(3, 3, MED_V8.MAX_DAYS, 1).setNumberFormat('0.0" ml"');
  sheet.getRange(3, 5, MED_V8.MAX_DAYS, 2).setNumberFormat('0.0" ml"');
  [120, 90, 155, 90, 150, 165].forEach(function(width, index) {
    sheet.setColumnWidth(index + 1, width);
  });
  sheet.setFrozenRows(2);
  sheet.setTabColor('#ED7D31');
  sheet.getRange('A2').setNote('실제로 투약 체크하여 새 병을 사용한 기록만 표시됩니다.');
}

function resetSheet_(ss, name, minRows, minColumns) {
  let sheet = ss.getSheetByName(name);
  if (!sheet) sheet = ss.insertSheet(name);
  if (sheet.getFilter()) sheet.getFilter().remove();

  const all = sheet.getRange(1, 1, sheet.getMaxRows(), sheet.getMaxColumns());
  all.breakApart();
  all.clear();
  all.clearDataValidations();
  all.clearNote();
  sheet.setConditionalFormatRules([]);
  sheet.setFrozenRows(0);
  sheet.setFrozenColumns(0);
  sheet.showRows(1, sheet.getMaxRows());
  sheet.showColumns(1, sheet.getMaxColumns());

  if (sheet.getMaxRows() < minRows) {
    sheet.insertRowsAfter(sheet.getMaxRows(), minRows - sheet.getMaxRows());
  }
  if (sheet.getMaxColumns() < minColumns) {
    sheet.insertColumnsAfter(sheet.getMaxColumns(), minColumns - sheet.getMaxColumns());
  }
  return sheet;
}

/** 기존 설정과 기록을 가능한 범위에서 보존 */
function readExistingData_(ss) {
  const result = { settings: {}, actions: {} };
  const settings = ss.getSheetByName(MED_V8.SETTINGS);

  if (settings) {
    settings.getDataRange().getValues().forEach(function(row) {
      const label = normalize_(row[0]);
      const value = row[1];
      if (!label) return;

      if (label.indexOf('1회주입량') !== -1) result.settings.dose = value;
      else if (label.indexOf('새약병용량') !== -1 || label.indexOf('한병용량') !== -1) {
        result.settings.capacity = value;
      } else if (
        label.indexOf('첫병실제잔량') !== -1 ||
        label.indexOf('첫병시작잔량') !== -1 ||
        label.indexOf('시작병잔량') !== -1
      ) result.settings.firstRemaining = value;
      else if (
        label.indexOf('기록시작일') !== -1 ||
        label === '시작일' ||
        label.indexOf('시작주월요일') !== -1
      ) result.settings.startDate = value;
      else if (label.indexOf('기록일수') !== -1) result.settings.days = value;
      else if (label.indexOf('생성주수') !== -1 && isFinite(Number(value))) {
        result.settings.days = Number(value) * 7;
      } else if (
        label.indexOf('최소사용단위') !== -1 ||
        label.indexOf('최소주입단위') !== -1
      ) result.settings.unit = value;
    });
  }

  const check = ss.getSheetByName(MED_V8.CHECK);
  if (!check || check.getLastRow() < 2) return result;

  const values = check.getDataRange().getValues();
  let headerRow = -1;
  let dateCol = -1;
  let processCol = -1;
  let doseCol = -1;
  let skipCol = -1;

  for (let r = 0; r < Math.min(values.length, 12); r++) {
    const labels = values[r].map(normalize_);
    const d = labels.indexOf('날짜');
    if (d < 0) continue;

    headerRow = r;
    dateCol = d;
    processCol = labels.indexOf('처리');
    doseCol = labels.findIndex(function(v) {
      return v === '투약체크' || v === '체크' || v === 'check';
    });
    skipCol = labels.findIndex(function(v) {
      return v === 'skip체크' || v === 'skip' || v === '스킵' || v === '건너뜀';
    });
    break;
  }

  if (headerRow < 0) return result;

  const timeZone = ss.getSpreadsheetTimeZone() || Session.getScriptTimeZone() || 'Asia/Seoul';

  for (let r = headerRow + 1; r < values.length; r++) {
    const date = values[r][dateCol];
    if (!(date instanceof Date) || isNaN(date.getTime())) continue;

    let action = '';
    if (processCol >= 0) {
      const v = String(values[r][processCol] || '').trim().toLowerCase();
      if (v === '투약') action = '투약';
      else if (v === 'skip' || v === '스킵' || v === '건너뜀') action = 'skip';
    }

    if (!action && doseCol >= 0) {
      const v = values[r][doseCol];
      if (v === true || String(v).trim() === '투약') action = '투약';
    }
    if (!action && skipCol >= 0) {
      const v = values[r][skipCol];
      if (v === true || String(v).trim().toLowerCase() === 'skip') action = 'skip';
    }

    if (action) {
      const key = Utilities.formatDate(date, timeZone, 'yyyy-MM-dd');
      result.actions[key] = action;
    }
  }
  return result;
}

function restoreChecks_(check, settings, actions) {
  const keys = Object.keys(actions || {});
  if (keys.length === 0) return;

  const startDate = settings.getRange('B5').getValue();
  const days = Number(settings.getRange('B6').getValue());
  if (!(startDate instanceof Date) || isNaN(startDate.getTime()) || !isFinite(days)) return;

  const output = Array.from({ length: MED_V8.MAX_DAYS }, function() {
    return ['', ''];
  });
  const startUtc = Date.UTC(startDate.getFullYear(), startDate.getMonth(), startDate.getDate());

  keys.forEach(function(key) {
    const parts = key.split('-').map(Number);
    if (parts.length !== 3 || parts.some(function(v) { return !isFinite(v); })) return;

    const dateUtc = Date.UTC(parts[0], parts[1] - 1, parts[2]);
    const offset = Math.round((dateUtc - startUtc) / 86400000);
    if (offset < 0 || offset >= Math.min(days, MED_V8.MAX_DAYS)) return;

    if (actions[key] === '투약') output[offset] = ['투약', ''];
    else if (actions[key] === 'skip') output[offset] = ['', 'skip'];
  });

  check.getRange(MED_V8.FIRST_ROW, 3, MED_V8.MAX_DAYS, 2).setValues(output);
}

function styleTitle_(range) {
  range
    .setBackground('#1F4E78')
    .setFontColor('#FFFFFF')
    .setFontWeight('bold')
    .setFontSize(16)
    .setHorizontalAlignment('center')
    .setVerticalAlignment('middle');
}

function styleHeader_(range) {
  range
    .setBackground('#D9EAF7')
    .setFontWeight('bold')
    .setHorizontalAlignment('center')
    .setVerticalAlignment('middle');
}

function quoteSheet_(name) {
  return "'" + String(name).replace(/'/g, "''") + "'";
}

function normalize_(value) {
  return String(value == null ? '' : value)
    .trim()
    .toLowerCase()
    .replace(/\s+/g, '')
    .replace(/[()（）]/g, '')
    .replace(/ml/g, '');
}

function numberOr_(value, fallback) {
  if (value === '' || value == null) return fallback;
  const n = Number(value);
  return isFinite(n) ? n : fallback;
}

function numberOrBlank_(value) {
  if (value === '' || value == null) return '';
  const n = Number(value);
  return isFinite(n) ? n : '';
}

function positiveNumberOr_(value, fallback) {
  const n = Number(value);
  return isFinite(n) && n > 0 ? n : fallback;
}

function dateOr_(value, fallback) {
  if (value instanceof Date && !isNaN(value.getTime())) {
    const d = new Date(value);
    d.setHours(0, 0, 0, 0);
    return d;
  }
  return fallback;
}

function integerBetween_(value, fallback, minimum, maximum) {
  const n = Math.round(Number(value));
  if (!isFinite(n)) return fallback;
  return Math.max(minimum, Math.min(maximum, n));
}
```

</details>

스크립트는 [MIT License](https://opensource.org/license/mit)로 공개한다. 자유롭게 복사하고 수정할 수 있지만, 의료적 정확성이나 특정 목적에 대한 적합성을 보증하지 않는다. 실제 기록을 시작하기 전에 반드시 사본에서 자신의 설정과 동작을 검증해야 한다.

설치 함수는 `약병 check`, `병 교체 정보`, `설정` 시트를 다시 구성한다. 기존 시트에 적용할 때는 반드시 사본을 먼저 만들어야 한다.

`onEdit(e)`는 사용자가 `투약`과 `skip`을 편집할 때 자동으로 실행되는 Apps Script의 단순 트리거다. 별도의 트리거를 추가로 설치할 필요는 없다. 여러 보호자가 거의 동시에 수정할 수 있는 환경이라면, 변경 직후 두 체크 상태와 최종 기록을 다시 확인하는 편이 안전하다.

## 사용 방법

설치가 끝나면 평소에는 `약병 check` 시트만 보면 된다.

1. 실제로 투약한 뒤 `투약 체크`를 선택한다.
2. 그날 투약하지 않았다면 `skip 체크`를 선택한다.
3. 상단 요약과 현재 병 남은 양을 확인한다.
4. 새 병을 실제로 사용한 기록은 `병 교체 정보`에서 확인한다.

미래 날짜를 미리 체크하면 잔량 계산에도 실제 투약처럼 반영된다. 따라서 계획 표시용으로 미리 체크하지 않고, **그날 실제로 투약했는지 확인한 뒤 기록하는 방식**을 권한다.

## 공개하면서 정한 원칙

이 도구를 공개하는 이유는 모든 사람에게 같은 투약 규칙을 권하기 위해서가 아니다. 아이를 돌보는 보호자가 각자 의료진에게 안내받은 계획을 덜 헷갈리게 기록하도록, 다시 만들 수 있는 출발점을 공유하기 위해서다.

그래서 다음 원칙을 함께 남긴다.

1. 코드보다 처방과 제품 안내가 우선이다.
2. 본문에 공개한 실제 사용값과 횟수를 그대로 사용하지 않는다.
3. 투약 직후 기록하고, 미래 투약을 미리 완료 처리하지 않는다.
4. 시트에는 최소한의 정보만 기록하고 공유 대상을 제한한다.
5. 계산 결과가 실제 약병과 다르면 계산을 믿지 말고 설정과 기록부터 다시 확인한다.

[CDC의 Medication Safety 안내](https://www.cdc.gov/medication-safety/about/index.html)도 어린이에게 약을 투여할 때 제품 라벨을 따르고, 이해되지 않는 내용은 의료진이나 약사에게 확인하라고 권한다.

Apps Script의 동작과 권한은 Google 공식 문서에서 확인할 수 있다.

- [Apps Script 단순 트리거](https://developers.google.com/apps-script/guides/triggers)
- [Apps Script 권한 부여](https://developers.google.com/apps-script/guides/services/authorization)
- [Google Docs·Sheets 개인정보 기본 안내](https://support.google.com/docs/answer/10381817)

## 작은 도구를 직접 만든다는 것

플랫폼이 부족해서 불편한 시대는 아닌 것 같다. 오히려 너무 많아서, 어디에 무엇을 기록했는지 다시 찾아야 하는 일이 늘었다.

이번에는 새로운 서비스를 하나 더 고르지 않았다. 이미 가족이 알고 있는 Google Sheets를 데이터베이스이자 화면으로 쓰고, AI와 함께 작성한 Apps Script로 우리에게 필요한 동작만 보탰다.

이 정도의 작은 도구는 이제 문제를 정확히 설명하고, 실제로 써보고, 위험한 경계를 걷어내는 과정을 반복하면 직접 만들 수 있다. 중요한 것은 무엇이든 앱으로 만드는 능력보다 **앱이 결정하면 안 되는 것까지 함께 정하는 능력**이었다.

필요한 것은 빠르게 만들되, 투약의 판단은 사람과 의료진에게 남겨둔다. 이번 도구를 공개하면서 가장 지키고 싶은 선이다.

---

`#아이투약기록` `#소아투약기록` `#어린이투약` `#아기투약기록` `#아이약기록` `#아이약관리` `#아기약관리` `#투약일지` `#투약기록표` `#복약기록` `#투약체크` `#투약관리표` `#아이주사기록` `#소아주사기록` `#주사투약기록` `#약병잔량` `#약병잔량관리` `#약병교체` `#가정투약관리` `#보호자투약공유` `#가족투약공유` `#육아앱` `#육아도구` `#구글스프레드시트` `#GoogleAppsScript` `#ChatGPT` `#AI코딩` `#무료공개`
