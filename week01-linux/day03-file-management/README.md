# Day 03 — Linux 파일 생성·복사·이동·삭제

> 작성일: 2026-08-02  
> 권장 학습 시간: 1시간 30분~2시간

오늘은 디렉터리 안에서 파일을 만들고, 복사하고, 이름을 바꾸고, 이동하고, 삭제하는 방법을 학습합니다.

## 오늘의 학습 목표

- [ ] `touch`로 빈 파일을 생성할 수 있다.
- [ ] `cp`로 파일을 복사할 수 있다.
- [ ] `mv`로 파일을 이동할 수 있다.
- [ ] `mv`로 파일 이름을 변경할 수 있다.
- [ ] `rm`으로 파일을 삭제할 수 있다.
- [ ] `rmdir`와 `rm -r`의 차이를 설명할 수 있다.
- [ ] 상대경로와 절대경로를 사용해 파일을 관리할 수 있다.
- [ ] 삭제 전에 현재 위치와 대상을 확인하는 습관을 익힌다.

오늘 사용할 핵심 명령어:

```bash
touch
cp
mv
rm
mkdir
rmdir
ls
pwd
```

---

## 1. Linux에서 파일이란?

Linux에서는 일반 문서뿐 아니라 설정 정보와 장치 정보 등 많은 것을 파일 형태로 다룹니다. 오늘은 일반적인 텍스트 파일 이름을 사용해 연습합니다.

```text
notes.txt
command.txt
practice.md
```

`.txt`, `.md` 같은 부분을 확장자라고 합니다. 다만 Linux에서는 확장자가 파일 형식을 강제로 결정하지 않으며, 파일을 구분하기 편하도록 이름에 붙인다고 이해하면 충분합니다.

---

## 2. `touch` — 빈 파일 만들기

빈 파일 생성:

```bash
touch test.txt
ls -l
```

여러 파일을 한 번에 생성:

```bash
touch file1.txt file2.txt file3.txt
ls -l
```

파일 크기가 `0`이라면 내용이 없는 빈 파일입니다.

### 기존 파일에 `touch`를 사용하면?

이미 존재하는 파일에 `touch`를 실행해도 내용이 삭제되지는 않습니다. 일반적으로 파일의 접근·수정 시간이 현재 시각으로 갱신됩니다.

```bash
touch test.txt
```

Day 03에는 우선 빈 파일을 만드는 용도로 익힙니다.

---

## 3. `cp` — 파일 복사하기

`cp`는 copy의 약자입니다.

기본 형식:

```bash
cp 원본파일 복사본파일
```

예시:

```bash
touch original.txt
cp original.txt copy.txt
ls -l
```

복사 후에도 원본은 그대로 남아 있고 새로운 복사본이 생깁니다.

다른 디렉터리로 복사:

```bash
mkdir backup
cp original.txt backup/
ls backup
```

복사하면서 이름 변경:

```bash
cp original.txt backup/original-backup.txt
ls -l backup
```

---

## 4. `mv` — 파일 이동하기

`mv`는 move의 약자입니다.

```bash
mv 파일명 이동할디렉터리/
```

예시:

```bash
touch report.txt
mkdir documents
mv report.txt documents/
ls
ls documents
```

현재 디렉터리에서는 `report.txt`가 사라지고 `documents` 안에서 보여야 합니다.

---

## 5. `mv` — 파일 이름 변경하기

Linux에서는 파일 이름을 변경할 때도 `mv`를 사용합니다.

```bash
mv 기존이름 새로운이름
```

예시:

```bash
touch old-name.txt
mv old-name.txt new-name.txt
ls
```

`mv`의 두 가지 사용 형태:

```text
mv 파일 디렉터리/     → 파일 이동
mv 기존이름 새이름    → 이름 변경
```

둘 다 파일의 경로를 바꾸는 작업이라는 점에서는 같습니다.

---

## 6. `rm` — 파일 삭제하기

```bash
rm 파일명
```

연습용 파일 삭제:

```bash
touch delete-me.txt
ls
rm delete-me.txt
ls
```

### 매우 중요한 주의사항

터미널에서 `rm`으로 삭제한 파일은 일반적으로 휴지통으로 이동하지 않아 복구하기 어려울 수 있습니다. 삭제 전에는 항상 다음 순서를 지킵니다.

```bash
pwd
ls
rm 정확한파일이름
```

```text
현재 위치 확인 → 파일 목록 확인 → 정확한 대상 확인 → 삭제
```

---

## 7. 삭제 전에 확인받기

`-i` 옵션을 사용하면 삭제 전에 확인 질문이 표시됩니다.

```bash
rm -i test.txt
```

예상 질문:

```text
rm: remove regular empty file 'test.txt'?
```

- `y`: 삭제 진행
- `n`: 삭제 취소

초보 단계에서는 중요한 파일을 다룰 때 `-i` 옵션을 활용하는 습관이 도움이 됩니다.

---

## 8. `rmdir`와 `rm -r`의 차이

### `rmdir`

비어 있는 디렉터리만 삭제할 수 있습니다.

```bash
mkdir empty
rmdir empty
```

디렉터리 안에 파일이 있으면 삭제되지 않습니다.

```bash
mkdir test-directory
touch test-directory/test.txt
rmdir test-directory
```

오류 예시:

```text
Directory not empty
```

### `rm -r`

디렉터리의 내부 내용까지 재귀적으로 삭제합니다.

```bash
rm -r 디렉터리이름
```

`rm -r`은 디렉터리 안의 파일까지 모두 삭제하므로 매우 조심해야 합니다. 오늘은 자신이 만든 연습용 디렉터리에서만 사용하고 먼저 확인 옵션과 함께 연습합니다.

```bash
pwd
ls 디렉터리이름
rm -ri 디렉터리이름
```

> 경로가 불명확하거나 시스템 디렉터리인 경우에는 실행하지 않습니다.

---

## 9. 덮어쓰기 주의하기

복사하거나 이동할 위치에 같은 이름의 파일이 있으면 기존 파일이 덮어써질 수 있습니다.

```bash
cp original.txt copy.txt
```

확인받으려면 `-i` 옵션을 사용합니다.

```bash
cp -i original.txt copy.txt
mv -i original.txt copy.txt
```

---

## 10. Day 03 실습 환경 만들기

홈 디렉터리 아래에 오늘 사용할 연습 디렉터리를 만듭니다.

```bash
cd ~
mkdir -p linux-practice/day03
cd linux-practice/day03
pwd
ls -la
```

이후 실습은 이 디렉터리에서 순서대로 진행합니다.

---

## 11. 단계별 실습

### 실습 1 — 파일 생성

```bash
touch original.txt notes.txt command.txt
pwd
ls -l
```

확인할 내용:

- 파일 세 개가 모두 생성되었는가?
- 파일 크기가 `0`인가?
- 현재 위치가 `day03`인가?

### 실습 2 — 파일 복사

```bash
cp original.txt copy.txt
mkdir backup
cp notes.txt backup/
cp command.txt backup/command-backup.txt
ls -l
ls -l backup
```

원본이 남아 있는지, `backup` 안에 복사본 두 개가 있는지 확인합니다.

### 실습 3 — 파일 이름 변경

```bash
mv copy.txt original-copy.txt
mv command.txt linux-command.txt
ls
```

기존 이름이 사라지고 새 이름이 보이는지 확인합니다.

### 실습 4 — 파일 이동

```bash
mkdir documents
mv notes.txt documents/
ls
ls documents
```

이동하면서 이름도 변경합니다.

```bash
mv original-copy.txt documents/copy-backup.txt
ls -l documents
```

### 실습 5 — 상대경로로 파일 관리

```bash
cd ~/linux-practice/day03/documents
pwd
cp copy-backup.txt ../backup/
ls ../backup
mv copy-backup.txt ..
ls
ls ..
```

여기서 `..`은 상위 디렉터리인 `day03`을 의미합니다.

### 실습 6 — 절대경로로 복사

사용자 이름을 확인합니다.

```bash
whoami
```

다음 명령의 `사용자이름`을 `whoami` 결과로 바꿔 실행합니다.

```bash
cp /home/사용자이름/linux-practice/day03/linux-command.txt /home/사용자이름/linux-practice/day03/documents/
ls /home/사용자이름/linux-practice/day03/documents
```

`~/linux-practice/...`처럼 `~`를 사용할 수도 있지만, `~`는 Shell이 홈 디렉터리 경로로 확장하는 기호입니다.

### 실습 7 — 파일 삭제

```bash
cd ~/linux-practice/day03
touch delete1.txt delete2.txt
pwd
ls
rm delete1.txt
rm -i delete2.txt
ls
```

두 번째 파일은 확인 질문에 응답해 삭제합니다.

### 실습 8 — 디렉터리 삭제

빈 디렉터리 삭제:

```bash
cd ~/linux-practice/day03
mkdir empty-directory
rmdir empty-directory
```

파일이 있는 연습용 디렉터리 삭제:

```bash
mkdir remove-practice
touch remove-practice/test.txt
rmdir remove-practice
pwd
ls remove-practice
rm -ri remove-practice
```

`rmdir`가 실패하는 이유를 확인한 뒤, 현재 위치와 내부 파일을 확인하고 `rm -ri`를 실행합니다.

---

## 12. Day 03 종합 미션

홈 디렉터리로 이동한 뒤 명령어만 사용해 다음 구조를 만듭니다.

```bash
cd ~
```

```text
day03-mission/
├── original/
│   ├── linux.txt
│   └── ssh.txt
├── backup/
│   ├── linux-backup.txt
│   └── ssh-backup.txt
└── trash/
    └── delete-me.txt
```

### 수행 조건

1. `day03-mission` 디렉터리를 만든다.
2. 그 안에 `original`, `backup`, `trash` 디렉터리를 만든다.
3. `original` 안에 `linux.txt`, `ssh.txt`를 만든다.
4. 두 파일을 `backup`에 복사한다.
5. 복사하면서 이름 뒤에 `-backup`을 붙인다.
6. `trash` 안에 `delete-me.txt`를 만든다.
7. `delete-me.txt`를 삭제한다.
8. 비어 있는 `trash` 디렉터리를 `rmdir`로 삭제한다.
9. 최종 구조를 `tree` 또는 `ls -R`로 확인한다.

예상 최종 구조:

```text
day03-mission/
├── backup/
│   ├── linux-backup.txt
│   └── ssh-backup.txt
└── original/
    ├── linux.txt
    └── ssh.txt
```

---

## 13. 도전 미션

명령어를 그대로 제공하지 않고 직접 생각해서 수행합니다.

1. `practice.txt` 파일을 만든다.
2. `practice.txt`를 `practice-copy.txt`로 복사한다.
3. `practice-copy.txt`의 이름을 `result.txt`로 변경한다.
4. `result.txt`를 `backup` 디렉터리로 이동한다.
5. 원본 `practice.txt`를 삭제한다.
6. `backup/result.txt`가 존재하는지 확인한다.

사용할 명령어:

```bash
touch
cp
mv
rm
mkdir
ls
```

---

## 14. 자가 테스트

답을 보기 전에 직접 명령어를 적고 말로 설명합니다.

1. 빈 파일을 생성하는 명령어는 무엇인가?
2. `original.txt`를 `copy.txt`로 복사하려면?
3. `old.txt`를 `new.txt`로 이름을 변경하려면?
4. `test.txt`를 `backup` 디렉터리로 이동하려면?
5. `delete.txt`를 삭제하려면?
6. `rmdir`와 `rm -r`의 차이는 무엇인가?
7. 왜 `rm` 명령어를 조심해야 하는가?
8. 삭제 전에 확인 질문을 표시하는 옵션은 무엇인가?
9. `cp`와 `mv`의 가장 큰 차이는 무엇인가?
10. 기존 파일의 내용을 지우지 않고 `touch`를 실행하면 주로 무엇이 바뀌는가?

<details>
<summary>자가 테스트 답안 보기</summary>

1. `touch 파일명`
2. `cp original.txt copy.txt`
3. `mv old.txt new.txt`
4. `mv test.txt backup/`
5. `rm delete.txt`
6. `rmdir`는 빈 디렉터리만 삭제하고, `rm -r`은 내부 내용까지 재귀적으로 삭제한다.
7. 일반적으로 휴지통을 거치지 않아 복구하기 어려울 수 있기 때문이다.
8. `-i`
9. `cp`는 원본을 남기고 복사본을 만들지만, `mv`는 원본의 경로나 이름을 바꾼다.
10. 파일의 접근·수정 시간이 현재 시각으로 갱신된다.

</details>

---

## 15. 오늘 반드시 기억할 핵심

```bash
touch 파일명
cp 원본 복사본
mv 파일 디렉터리/
mv 기존이름 새로운이름
rm 파일명
rmdir 빈디렉터리
rm -r 디렉터리
```

삭제 전 안전 확인:

```bash
pwd
ls
rm -i 삭제할파일
```

특히 `cp`와 `mv`의 차이, 그리고 `mv`가 이동과 이름 변경에 모두 사용된다는 점을 확실히 익힙니다.

---

## 16. 학습 기록

실습 후 직접 작성합니다.

```text
오늘 새롭게 알게 된 것:

cp와 mv의 차이를 내 말로 설명:

가장 자주 사용한 명령어:

삭제 전에 확인한 내용:

실행 중 발생한 오류:

오류를 해결한 방법:

종합 미션 완료 여부:
```
