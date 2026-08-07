# Confirm Examinee Domain

대상:

- `Data/UBerSoft.EMR.Data.ConfirmExaminee`
- `Data/UBerSoft.EMR.Data.ExamineeInquiry`
- `Data/UBerSoft.EMR.Data/Controllers/Users/Concreate/PatientController.cs`
- `Data/UBerSoft.EMR.Data.ViewModels/Users/PatientViewModel.cs`
- `Data/UBerSoft.EMR.Data.Models/Users/Insurances/*.cs`

## 1. 목적

수진자조회는 환자의 주민등록번호와 이름으로 국민건강보험공단/의료급여 자격관리 시스템에 현재 자격을 조회하고, 그 결과를 EMR의 환자 보험 이력으로 저장하는 기능이다.

저장된 보험 이력은 이후 다음 업무에서 사용된다.

```text
환자 등록/접수
-> 현재 보험 자격 선택
-> 진료비 본인부담 계산
-> 의료급여 진료확인번호 관리
-> 산정특례/차상위/지원금 계산
-> SAM 청구 명세서 생성
-> 수납/영수증/환자 안내
```

핵심은 수진자조회 응답 원문을 그대로 쓰는 것이 아니라, 다음 4개 저장 모델로 정규화하는 것이다.

| 저장 묶음 | 역할 |
|---|---|
| `Insurance` | 건강보험/의료급여/일반 자격 이력의 중심 |
| `MedicalAid` | 의료급여 종별, 보장기관, 선택기관, 지원금 잔액 |
| `SpecialCase` | 산정특례, 차상위, 조산아, 자립준비 청년 등 등록 정보 |
| `AppointHospital` | 의료급여 선택기관 지정 정보 |

## 2. 현재 소스 기준 구현 구조

현재 실제 사용 경로:

```text
PatientViewModel.ConfirmExamineeTask
-> PatientController.ConfirmExaminee
-> ConfirmExaminee.ConfirmExaminee.SetQualify
-> WebService.dll WS_AddParam / WS_Qualify
-> ConfirmExaminee.GetQualify
-> PatientController.ConfirmExamineeInternal
-> InsuranceViewModel 생성
-> 기존 보험 이력과 병합
-> UpsertInsurance
```

보조/대체 구현:

- `UBerSoft.EMR.Data.ExamineeInquiry`는 C#으로 작성된 메시지 모델/로직이다.
- 현재 `PatientController`에서는 해당 C# 경로가 주석 처리되어 있고, VB 프로젝트 `UBerSoft.EMR.Data.ConfirmExaminee`를 사용한다.
- 신규 개발에서는 C# `ExamineeInquiry`의 메시지 타입 구조를 기준으로 API 계약을 설계하고, 현재 `ConfirmExamineeInternal`의 저장 매핑 규칙을 따라가면 된다.

## 3. 메시지 타입

| 메시지 | 방향 | 목적 | 현재 업무 의미 |
|---|---|---|---|
| `M1` | EMR -> 공단 | 수진자 자격조회 요청 | 접수 전/환자 등록 시 보험자격 확인 |
| `M2` | 공단 -> EMR | 수진자 자격조회 결과 | 보험 이력, 산정특례, 의료급여 정보 저장 |
| `M3` | EMR -> 공단 | 의료급여 진료확인 승인 요청 | 의료급여 수납/진료확인번호 발급 |
| `M4` | 공단 -> EMR | 의료급여 진료확인 승인 결과 | 진료확인번호, 건강생활유지비/산전지원금 잔액 반영 |
| `M5` | EMR -> 공단 | 의료급여 진료확인 취소 요청 | 승인 취소 |
| `M6` | 공단 -> EMR | 의료급여 진료확인 취소 결과 | 취소 여부와 잔액 반영 |

## 4. M1 수진자 자격조회 요청 필드

생성 위치:

- `ConfirmExaminee.SetQualify`
- `ExamineeInquiry.Logic.CreateM1`

| 필드명 | 외부 키 | 뜻 | 생성 규칙 |
|---|---|---|---|
| `수진자주민등록번호` | `sujinjaJuminNo` | 조회할 환자의 주민등록번호 | 하이픈 제거 후 전송. VB 경로는 전달받은 값을 그대로 사용 |
| `의료급여기관기호` | `ykiho` | 요양기관기호 | 운영 서버는 병원 코드, 테스트 URL이면 테스트 코드 사용 |
| `수진자성명` | `sujinjaJuminNm` | 환자 이름 | 환자 마스터의 이름 |
| `idYN` | `idYN` | 실명/식별 확인 여부 | `idchk == true`이면 `Y`, 아니면 `N` |
| `진료일자` | `diagDt` | 조회 기준일 | C# 모델은 현재일, VB 경로는 현재 미전송 |
| `건강보험증번호` | `hiCardNo` | 건강보험증 번호 | 선택 입력 |
| `생년월일` | `birthDay` | 생년월일 | 선택 입력 |
| `화면클라이언트고유값` | `clientInfo` | 클라이언트 식별값 | C# 모델은 IP, VB 경로는 빈값 |
| `메시지타입` | `msgType` | 요청 메시지 구분 | 고정 `M1` |

개발 기준:

- 접수 화면에서 최소 필수값은 환자명, 주민등록번호, 요양기관기호이다.
- 수진자조회 옵션이 꺼져 있으면 조회하지 않고 기존 보험 이력을 사용한다.
- 병원 인증/자동 비밀번호 설정 로직이 선행되어야 한다.

## 5. M2 수진자 자격조회 결과 필드

수신 위치:

- `ConfirmExaminee.GetQualify`
- `ExamineeInquiry.Model.InquiryM2`

### 기본 자격 필드

| 필드명 | 외부 키 | 뜻 | 저장 위치 |
|---|---|---|---|
| `서버메시지코드` | `messageCode` | 조회 결과 코드 | `Insurance.ExamineeMessage`와 오류 처리 |
| `서버메시지` | `message` | 조회 결과 메시지 | `Insurance.ExamineeMessage` |
| `수진자주민등록번호` | `sujinjaJuminNo` | 조회된 수진자 주민번호 | 환자 검증용, 별도 보험 저장은 하지 않음 |
| `수진자성명` | `sujinjaJuminNm` | 조회된 수진자 성명 | 환자 검증용 |
| `자격여부` | `qlfType` | 건강보험/의료급여 자격 구분 | `Insurance.InsurnerType`, `MedicalAid.Type` |
| `자격취득일` | `qlfChwidukDt` | 자격 시작일 | `Insurance.FromApplyDate` |
| `세대주성명` | `sedaejuNm` | 건강보험 가입자/의료급여 세대주 성명 | `Insurance.MemberName` |
| `보장기관기호/사업장기호` | `protAdminSym` | 의료급여 보장기관 또는 건강보험 사업장 성격 코드 | 의료급여이면 `MedicalAid.AgencyCode` |
| `시설기호/증번호` | `asylmSym` | 건강보험 증번호 또는 시설기호 | 건강보험이면 `Insurance.InsuranceNumber` |
| `급여제한일자/건강보험상실일자` | `payRestricDt` | 급여 제한일 또는 상실일 | 조건에 따라 `Insurance.ToApplyDate`, 선택기관 시작일 |
| `건강보험자격상실처리일자` | `sangsilProcDt` | 건강보험 자격 상실 처리일 | 건강보험이면 `Insurance.ToApplyDate` |
| `급여제한여부` | `qlfRestrictCd` | 무자격/체납/사망 등 급여제한 코드 | `Insurance.InsuranceLimitType.Code` |
| `출국자여부` | `dprtYn` | 출국으로 인한 급여정지 여부 | `Insurance.IsExitCountry` |
| `국적구분` | `ntntType` | 내국인/외국인/재외국민 | 지역가입자이면 `Insurance.NationalityCode.Code` |

자격여부 코드:

| 코드 | 뜻 | EMR 보험구분 |
|---:|---|---|
| `1` | 지역가입자 | 건강보험 |
| `2` | 지역세대원 | 건강보험 |
| `4` | 임의계속직장가입자 | 건강보험 |
| `5` | 직장가입자 | 건강보험 |
| `6` | 직장피부양자 | 건강보험 |
| `7` | 의료급여 1종 | 의료급여, `MedicalAid.Type.Code = 1` |
| `8` | 의료급여 2종 | 의료급여, `MedicalAid.Type.Code = 2` |
| `99` 또는 없음 | 없음 | 일반/무자격 처리 |

오류 처리:

- 정상 코드는 `IWS10001`.
- 메시지 코드가 있고 `IWS10001`이 아니면 사용자 오류로 중단한다.
- `자격취득일`이 날짜로 파싱되지 않으면 정상 자격으로 보지 않는다.

### 의료급여 필드

| 필드명 | 외부 키 | 뜻 | 저장 위치 |
|---|---|---|---|
| `본인부담여부` | `sbrdnType` | 의료급여 본인부담 코드, 선택기관/임산부/18세미만 등 | `MedicalAid.PatientCostCode.Code` |
| `건강생활유지비잔액` | `cfhcRem` | 의료급여 건강생활유지비 잔액 | `MedicalAid.HalthSupportMoney` |
| `의료급여산전지원금잔액` | `pregRemAmt` | 의료급여 산전지원금 잔액 | `MedicalAid.PrevBirthSupportMoney` |
| `선택기관기호1` | `ykiho1` | 1차 선택기관 기호 | `MedicalAid.AppointHospitals[1].HospitalCode` |
| `선택기관기호2` | `ykiho2` | 2차 선택기관 기호 | `MedicalAid.AppointHospitals[2].HospitalCode` |
| `선택기관기호3` | `ykiho3` | 3차 선택기관 기호 | `MedicalAid.AppointHospitals[3].HospitalCode` |
| `선택기관기호4` | `ykiho4` | 4차 선택기관 기호 | `MedicalAid.AppointHospitals[4].HospitalCode` |
| `선택기관이름1` | `yoyangNm1` | 1차 선택기관명 | `AppointHospital.HospitalName` |
| `선택기관이름2` | `yoyangNm2` | 2차 선택기관명 | `AppointHospital.HospitalName` |
| `선택기관이름3` | `yoyangNm3` | 3차 선택기관명 | `AppointHospital.HospitalName` |
| `선택기관이름4` | `yoyangNm4` | 4차 선택기관명 | `AppointHospital.HospitalName` |
| `동일성분의약품제한자` | `disRegPrson7` | 동일성분 의약품 제한 기간 | `MedicalAid.FromSameDrugLimiter`, `ToSameDrugLimiter` |

의료급여 변환 규칙:

- `qlfType == 7`이면 의료급여 1종.
- `qlfType == 8`이면 의료급여 2종.
- 장애여부가 `Y`이고 의료급여 2종이면 `MedicalAid.Type.Code = 8`로 바꾼다.
- `sbrdnType`이 `M001`, `M002`, `B001`, `B002`이면 `payRestricDt`를 선택기관 지정 시작일로 사용한다.

### 차상위/산정특례/지원 대상 필드

수진자조회 응답의 여러 `disRegPrson*` 필드는 `SpecialCase`로 저장한다.

| 필드명 | 외부 키 | 뜻 | 저장 SpecialCase |
|---|---|---|---|
| `산정특례희귀등록대상자` | `disRegPrson2` | 희귀 산정특례 | `산정특례_희귀등록대상자` |
| `차상위대상자` | `disRegPrson3` | 차상위 본인부담 경감 | `차상위대상자` |
| `산정특례암등록대상자` | `disRegPrson4` | 암 산정특례 | `산정특례_암_등록대상자` |
| `산정특례화상등록대상자` | `disRegPrson5` | 화상 산정특례 | `산정특례_화상_등록대상자` |
| `당뇨병요양비대상자등록일` | `disRegPrson6` | 당뇨병 요양비 등록일 | `Insurance.DiabetesRegisterDate` |
| `자가도뇨카테타대상자` | `disRegPrson8` | 자가도뇨 카테타 대상 등록일 | `Insurance.SelfKatheter` |
| `산정특례결핵등록대상자` | `disRegPrson9` | 결핵 산정특례 | `산정특례_결핵_등록대상자` |
| `산정특례극희귀등록대상자` | `disRegPrson10` | 극희귀 산정특례 | `산정특례_극희귀_등록대상자` |
| `산정특례상세불명희귀등록대상자` | `disRegPrson11` | 상세불명 희귀 산정특례 | `산정특례_상세불명희귀_등록대상자` |
| `요양기관별결핵등록대상자` | `disRegPrson12` | 요양기관별 결핵 산정특례 | 현재 저장 로직 주석 처리 |
| `산정특례중복암2` | `disRegPrson13` | 중복암 산정특례 | `산정특례_중복암_등록대상자` |
| `산정특례중복암3` | `disRegPrson14` | 중복암 산정특례 | `산정특례_중복암_등록대상자` |
| `산정특례중복암4` | `disRegPrson15` | 중복암 산정특례 | `산정특례_중복암_등록대상자` |
| `산정특례중복암5` | `disRegPrson16` | 중복암 산정특례 | `산정특례_중복암_등록대상자` |
| `산정특례중증치매등록대상자` | `disRegPrson17` | 중증치매 산정특례 | `산정특례_중증치매_등록대상자` |
| `산정특례중증난치등록대상자` | `disRegPrson18` | 중증난치 산정특례 | `산정특례_중증난치질환_등록대상자` |
| `기타염색체이상질환등록대상자` | `disRegPrson19` | 기타염색체 이상질환 산정특례 | `산정특례_기타염색체이상질환_등록대상자` |
| `잠복결핵등록대상자` | `disRegPrson20` | 잠복결핵 산정특례 | `산정특례_잠복결핵_등록대상자` |
| `조산아및저체중아` | `preInfant` | 조산아/저체중 출생아 등록 | `조산아및저체중출생아_등록대상자` |
| `자립준비청년대상자` | `slfPreprPrson` | 자립준비 청년 대상 | `자립준비_청년_대상자` |

SpecialCase 원문 파싱 규칙:

| 유형 | 원문 구성 | 저장 필드 |
|---|---|---|
| 차상위 | 특정기호 4 + 시작일 8 + 종료일 8 + 구분 1 | `SpecificCode`, `FromApplyDate`, `ToApplyDate` |
| 화상/결핵 | 특정기호 4 + 등록번호 15 + 등록일 8 + 종료일 8 | `SpecificCode`, `RegisterNumber`, `FromApplyDate`, `ToApplyDate` |
| 희귀/극희귀/상세불명희귀/중증난치/기타염색체/잠복결핵 | 특정기호 4 + 등록번호 15 + 등록일 8 + 종료일 8 + 상병코드 10 + 상병일련번호 | `SpecificCode`, `RegisterNumber`, `FromApplyDate`, `ToApplyDate`, `DiseaseCode` |
| 암/중복암 | 특정기호 4 + 등록번호 15 + 등록일 8 + 종료일 8 + 상병코드 10 + 상병일련번호 + 등록구분 1 | 위 필드 + `RegisterType` |
| 중증치매 | 특정기호, 상병코드, 등록번호, 시작/상실/차수 기간, 사전승인일수 | 현재 저장은 공통 `SpecialCaseMethod` 기준으로 최소 필드만 사용 |

차상위/지원금 변환:

- 차상위 대상자가 있으면 `GovernmentInjuryType.Code`에 차상위 특정기호를 넣는다.
- 차상위 특정기호 `C`, `E`, `F`는 환자비용 계산에서 차상위 판정에 사용된다.
- 장애여부가 `Y`이고 차상위 코드가 `E`이면 `F`로 변경한다.
- 희귀/극희귀/상세불명희귀 특정기호가 `H`이면 `GovernmentInjuryType.Code = H`로 저장해 지원금 계산에 사용한다.

### 기타 안내/판정 필드

| 필드명 | 외부 키 | 뜻 | 저장 위치/사용 |
|---|---|---|---|
| `장애여부` | `obstYn` | 장애 여부 | 의료급여 2종 장애, 차상위 장애 전환 |
| `장애인등록일자` | `obstRegDt` | 장애 등록일 | 현재 직접 저장 안 함 |
| `노인틀니상악/하악` | `dentTop`, `dentBottom` | 노인틀니 대상자 정보 | 현재 직접 저장 안 함 |
| `임플란트대상자정보1/2` | `dentImpl1`, `dentImpl2` | 임플란트 대상자 정보 | 현재 직접 저장 안 함 |
| `당뇨병요양비대상자유형` | `diabetesCd` | 당뇨 유형 | `Insurance.DiabetesType.Code` |
| `요양병원입원여부` | `mdcareHsptHsptzYn` | 타 요양병원 입원 중 여부 | `Insurance.IsMdcareHsptHsptzYn` |
| `요양병원기관기호` | `mdcareHsptAdminSym` | 입원 요양병원 기호 | `Insurance.MdcareHsptAdminSym` |
| `비대면진료대상정보` | `nftfDiagTgtInfo` | 산간벽지/장애인/장기요양 비대면 대상 여부 | `IsNftfSecludedPlace`, `IsNftfDisabled`, `IsNftfLongTermCare` |
| `방문진료본인부담경감대상자` | `visMcareTgtInfo` | 방문진료 본인부담 경감 대상 정보 | `Insurance.VisMcareTgtInfo` |

## 6. 저장해야 할 필드 정리

### `Insurance`

| 필드 | 뜻 | 왜 필요한가 |
|---|---|---|
| `InsurnerType` | 건강보험/의료급여/일반/건강보험_100 구분 | 접수 보험 유형, 비용계산, 청구서 분리 기준 |
| `FromApplyDate` | 자격취득일 | 진료일 기준 현재 보험 선택 |
| `ToApplyDate` | 자격상실일/급여제한일 | 이력 관리, 무자격 안내 |
| `MemberName` | 가입자/세대주 이름 | SAM 명세서 수진자 정보 |
| `InsuranceNumber` | 건강보험 증번호/시설기호 | SAM 명세서 수진자 보험번호 |
| `GovernmentInjuryType.Code` | 공상등 구분, 차상위, 긴급복지, 희귀지원 | 본인부담/지원금/SAM 공상등구분 |
| `InsuranceLimitType.Code` | 급여제한 코드 | 무자격/체납/사망 등 안내와 접수 경고 |
| `IsExitCountry` | 출국자 여부 | 무자격 처리와 접수 경고 |
| `IsNftfSecludedPlace` | 비대면 산간벽지 대상 | 비대면 진료 판단 |
| `IsNftfDisabled` | 비대면 장애인 대상 | 비대면 진료 판단 |
| `IsNftfLongTermCare` | 비대면 장기요양 대상 | 비대면 진료 판단 |
| `VisMcareTgtInfo` | 방문진료 본인부담 경감 대상 정보 | 방문진료 수가/본인부담 판단 |
| `IsDisableJangru` | 장루/요루 장애인 | 장루요루 본인부담 특례 |
| `SpecialCases` | 산정특례/차상위 목록 | 진료비 특례 계산, SAM 특정내역 |
| `DiabetesRegisterDate` | 당뇨병 요양비 등록일 | 환자 안내/요양비 대상 관리 |
| `DiabetesType.Code` | 당뇨병 요양비 대상자 유형 | 당뇨 요양비 구분 |
| `SelfKatheter` | 자가도뇨 카테타 대상자 등록일 | 환자 특수 대상 관리 |
| `NationalityCode.Code` | 국적구분 | 지역가입자 국적 정보 |
| `IsMdcareHsptHsptzYn` | 요양병원 입원여부 | 진료의뢰서/전액부담 안내 |
| `MdcareHsptAdminSym` | 입원 요양병원 기호 | 요양병원 입원 안내/청구 검증 |
| `ExamineeMessage` | 조회 결과 메시지 | 조회 결과 안내/변경 없음 안내 |

### `MedicalAid`

| 필드 | 뜻 | 왜 필요한가 |
|---|---|---|
| `AgencyCode` | 보장기관기호 | 의료급여 SAM 명세서 `MedicalAidAgencyCode` |
| `Type.Code` | 의료급여 종별 | 의료급여 본인부담 계산, SAM 의료급여 종별 |
| `PatientCostCode.Code` | 의료급여 본인부담 코드 | 면제/선택기관/임산부/18세미만 계산, SAM `MT018` |
| `HalthSupportMoney` | 건강생활유지비 잔액 | 의료급여 수납 승인/지원금 차감 |
| `PrevBirthSupportMoney` | 산전지원금 잔액 | 임산부/출산전 진료비 승인 |
| `FromSameDrugLimiter` | 동일성분 의약품 제한 시작일 | 처방/약제 관리 경고 |
| `ToSameDrugLimiter` | 동일성분 의약품 제한 종료일 | 처방/약제 관리 경고 |
| `AppointHospitals` | 선택의료급여기관 목록 | 선택기관 일치 여부, 본인부담 면제, SAM `MT019` 관련 |

의료급여 종별 코드:

| 코드 | 뜻 |
|---|---|
| `1` | 의료급여 1종 |
| `2` | 의료급여 2종 |
| `4` | 행려 |
| `6` | 2종 장애인의 2차 의료급여 |
| `8` | 2종 장애인의 1차 의료급여 |
| `N` | 노숙인 1종 |

### `SpecialCase`

| 필드 | 뜻 | 왜 필요한가 |
|---|---|---|
| `SpecialCaseCode.Code` | 산정특례/차상위 종류 | 어떤 계산 서비스를 적용할지 결정 |
| `FromApplyDate` | 특례 시작일 | 진료일 기준 특례 유효성 판단 |
| `ToApplyDate` | 특례 종료일 | 진료일 기준 특례 유효성 판단 |
| `SpecificCode.Code` | 특정기호 4자리 | 본인부담률, SAM `MT002`, `MT014` |
| `DiseaseCode` | 특례 상병코드 | 주상병과 특례 매칭 |
| `RegisterNumber` | 산정특례 등록번호 | SAM `MT014`, 출력/청구 검증 |
| `RegisterType.Code` | 암 등록구분 등 | 암/중복암 구분 |
| `ExamineeValue` | 원문 응답값 | 파싱 검증/추적용 |

### `AppointHospital`

| 필드 | 뜻 | 왜 필요한가 |
|---|---|---|
| `ApplyDate` | 선택기관 지정 시작일 | 선택기관 유효기간 판단 |
| `ClassNumber` | 1~4차 선택기관 순번 | 선택기관 표시/검증 |
| `HospitalCode` | 선택기관 요양기관기호 | 현재 병원과 일치 여부 확인 |
| `HospitalName` | 선택기관명 | 접수 안내/경고 |

## 7. 보험 이력 병합 규칙

파일: `PatientViewModel.ConfirmExamineeTask`, `MergeInsurance`

규칙:

- 조회 결과가 없으면 기존 보험을 변경하지 않는다.
- 환자 저장 전이면 조회 결과를 그대로 보험 목록으로 사용한다.
- 이미 저장된 환자이면 기존 보험 이력과 조회 결과를 병합한다.
- 동일한 보험인지 비교할 때 다음 값을 본다.
  - 자격취득일
  - 보험구분
  - 가입자/세대주명
  - 증번호
  - 공상등 구분
  - 급여제한 코드
  - 자격상실일
  - 출국자 여부
  - 국적구분
  - 산정특례 목록
  - 의료급여 보장기관/종별/본인부담코드/지원금/선택기관
- 동일하지 않은 신규 조회 결과는 `FromApplyDate`가 같은 기존 이력을 대체한다.
- `FromApplyDate`가 다르면 새 이력으로 추가한다.

현재 보험 선택 규칙:

```text
진료일 이하의 FromApplyDate 중 가장 최신 이력 선택
```

주의:

- 현재 소스는 `ToApplyDate`를 현재 보험 선택 조건에서 제외한다.
- 이유는 무자격자도 건강보험 이력으로 잡히면서 `ToApplyDate`가 들어올 수 있기 때문이다.
- 신규 구현에서도 무자격/급여제한을 별도 플래그로 판단해야 하며, 단순히 종료일만으로 일반 처리하면 안 된다.

## 8. M3 의료급여 진료확인 승인 요청

생성 위치:

- `ExamineeInquiry.Logic.CreateM3`
- VB 프로젝트의 `ConfirmApproval`/`Modes.ExamineeApproval` 계열

의료급여 진료확인은 의료급여 수급권자의 실제 진료 후 본인부담금, 건강생활유지비, 기관부담금 등을 승인받고 `진료확인번호`를 확보하는 절차이다.

| 필드명 | 외부 키 | 뜻 | 생성 규칙 |
|---|---|---|---|
| `수진자성명` | `sujinjaJuminNm` | 환자 이름 | 환자 정보 |
| `수진자주민등록번호` | `sujinjaJuminNo` | 환자 주민번호 | 환자 정보 |
| `진료일자` | `diagDt` | 진료일 | 진료/수납 기준일 |
| `의료급여기관기호` | `ykiho` | 요양기관기호 | 병원 코드 |
| `건강보험증번호` | `hiCardNo` | 의료급여 카드/증번호 성격 | 조회 결과 또는 환자 보험정보 |
| `진료형태` | `diagType` | 입원/외래 구분 | 병의원 외래는 보통 `2` |
| `입원일수` | `payDdCnt` | 입원일수 또는 외래 1 | 외래는 `1` |
| `투약일수` | `tuyakDdCnt` | 투약일수 | 처방 기준 |
| `본인일부부담금` | `selfPartBrdnAmt` | 의료급여 본인부담금 | 진료비 계산 결과 |
| `건강생활유지비청구액` | `cfhcDmdAmt` | 건강생활유지비에서 차감할 금액 | 수납 계산 결과 |
| `기관부담금` | `adminBrdnAmt` | 기관 부담금 | 청구/수납 계산 결과 |
| `주상병기호` | `mainSickSym` | 주상병 코드 | 진료 상병 |
| `처방전교부번호` | `prscGnoAdmin` | 원외처방전 번호 | 원외처방 있으면 입력 |
| `본인부담여부` | `sbrdnType` | 의료급여 본인부담 코드 | 환자 보험정보 `MedicalAid.PatientCostCode` |
| `타기관의뢰여부` | `otherRequestYn` | 타기관 의뢰 여부 | `Y/N` |
| `진료확인번호` | `cfhcCfrNo` | 기존 진료확인번호 | 재승인/수정 성격이면 입력 |
| `진료과코드` | `diagItem` | 진료과 | 진료실/의사 진료과 |
| `처방전발급유무` | `prscGnoYn` | 처방전 발급 여부 | 처방전교부번호 있으면 `Y` |
| `퇴원구분코드` | `diagOutCode` | 퇴원 구분 | 입원 업무에서 사용 |
| `비급여총금액` | `pregSumAmt` | 비급여 총액 | 진료비 계산 결과 |
| `출산전진료비청구액` | `pregDmndAmt` | 산전지원금 청구액 | 임산부/산전지원 사용 시 |
| `진료의뢰의료급여기관기호` | `diagReqYkiho` | 의뢰기관 기호 | 선택기관/의뢰 진료 시 |

## 9. M4 의료급여 진료확인 승인 결과

| 필드명 | 외부 키 | 뜻 | 저장/사용 |
|---|---|---|---|
| `승인여부` | `admType` | 승인 결과 | `03`이면 승인 |
| `진료확인번호` | `cfhcCfrNo` | 공단에서 발급한 진료확인번호 | 진료 `MedicalAidConfirmNumber`로 저장해야 함 |
| `본인일부부담금` | `selfPartBrdnAmt` | 승인된 본인부담금 | 수납/검증 |
| `건강생활유지비청구액` | `cfhcDmdAmt` | 승인된 건강생활유지비 청구액 | 수납/지원금 차감 |
| `건강생활유지비잔액` | `cfhcRem` | 승인 후 잔액 | 환자 의료급여 정보 갱신 |
| `출산전진료비청구액` | `pregDmndAmt` | 승인된 산전지원 청구액 | 수납/지원금 차감 |
| `의료급여산전지원금잔액` | `pregRemAmt` | 승인 후 산전지원금 잔액 | 환자 의료급여 정보 갱신 |
| `서버메시지코드` | `messageCode` | 결과 코드 | 오류 처리 |
| `서버메시지` | `message` | 결과 메시지 | 사용자 안내 |

개발 기준:

- 의료급여 진료는 청구 전에 진료확인번호가 필요하다.
- 보험청구 검증에서도 의료급여이고 `MedicalAidConfirmNumber`가 없으면 오류로 본다.
- 승인 후 잔액은 환자 보험정보 또는 수납 당시 스냅샷에 반영해야 한다.

## 10. M5/M6 진료확인 취소

M5 요청:

| 필드명 | 외부 키 | 뜻 |
|---|---|---|
| `수진자주민등록번호` | `sujinjaJuminNo` | 환자 주민번호 |
| `진료일자` | `diagDt` | 승인받은 진료일 |
| `의료급여기관기호` | `ykiho` | 병원 코드 |
| `진료확인번호` | `cfhcCfrNo` | 취소할 승인 번호 |
| `메시지타입` | `msgType` | 고정 `M5` |

M6 결과:

| 필드명 | 외부 키 | 뜻 | 사용 |
|---|---|---|---|
| `취소여부` | `cnclType` | 취소 결과 | `06`이면 취소 성공 |
| `진료확인번호` | `cfhcCfrNo` | 취소 대상 번호 | 진료와 매칭 |
| `건강생활유지비잔액` | `cfhcRem` | 취소 후 잔액 | 의료급여 잔액 복원 |
| `의료급여산전지원금잔액` | `pregRemAmt` | 취소 후 산전지원금 잔액 | 의료급여 잔액 복원 |
| `서버메시지코드` | `messageCode` | 결과 코드 | 오류 처리 |
| `서버메시지` | `message` | 결과 메시지 | 사용자 안내 |

## 11. 접수/진료/청구에서 사용하는 방식

접수:

- 환자 조회 또는 신규 등록 시 수진자조회를 실행한다.
- 조회 성공 시 보험 이력을 병합 저장한다.
- 접수일 기준 `CurrentInsurance`를 가져와 접수 보험을 결정한다.
- 출국자/급여제한자/요양병원 입원자는 접수 화면에 경고를 보여준다.

비용 계산:

- `InsurnerType`이 건강보험/의료급여/일반 계산 분기 기준이다.
- `GovernmentInjuryType`은 공상, 차상위, 긴급복지, 희귀지원 계산에 사용된다.
- `MedicalAid.Type`, `PatientCostCode`, `AppointHospitals`는 의료급여 본인부담 계산에 사용된다.
- `SpecialCases`는 산정특례 본인부담률과 지원금 계산에 사용된다.

청구:

- 건강보험/의료급여는 청구서가 분리된다.
- `MemberName`, `InsuranceNumber`, `MedicalAid.AgencyCode`, `MedicalAid.Type`이 SAM 명세서 수진자/의료급여 필드로 들어간다.
- 산정특례는 `MT002`, `MT014` 등 특정내역 생성에 영향을 준다.
- 의료급여 `PatientCostCode`는 `MT018`, 진료확인번호는 `MT019` 생성과 연결된다.

## 12. 신규 개발 권장 구조

권장 모듈:

| 모듈 | 책임 |
|---|---|
| `ExamineeInquiryClient` | 공단 API/DLL 호출, M1~M6 송수신 |
| `ExamineeMessageMapper` | 외부 메시지와 내부 DTO 변환 |
| `InsuranceQualificationService` | M2 결과를 `Insurance` 이력으로 변환 |
| `InsuranceHistoryService` | 기존 보험 이력 병합/중복 판단 |
| `MedicalAidApprovalService` | M3~M6 진료확인 승인/취소 |
| `InsuranceWarningService` | 출국자/급여제한/요양병원 입원/선택기관 경고 |

DB 저장 권장:

- `PatientInsurance`
- `PatientMedicalAid`
- `PatientSpecialCase`
- `PatientAppointHospital`
- `MedicalAidConfirm` 또는 진료 테이블의 `MedicalAidConfirmNumber`
- 원문 추적용 `ExamineeInquiryLog`

원문 로그에는 다음을 남기는 것이 좋다.

| 필드 | 뜻 |
|---|---|
| `PatientId` | 환자 |
| `RequestType` | M1/M3/M5 |
| `ResponseType` | M2/M4/M6 |
| `RequestAt` | 요청 시각 |
| `ResponseAt` | 응답 시각 |
| `HospitalCode` | 요양기관기호 |
| `MessageCode` | 응답 코드 |
| `Message` | 응답 메시지 |
| `RawRequest` | 요청 원문 |
| `RawResponse` | 응답 원문 |

## 13. 개발 순서

1. 환자명/주민번호/병원코드로 M1 수진자조회 요청을 구현한다.
2. M2 응답을 DTO로 안정적으로 파싱한다.
3. 자격여부를 건강보험/의료급여/일반으로 변환한다.
4. `Insurance`, `MedicalAid`, `SpecialCase`, `AppointHospital` 저장 모델을 만든다.
5. 기존 보험 이력 병합 규칙을 구현한다.
6. 접수일 기준 현재 보험 선택 규칙을 구현한다.
7. 출국자/급여제한/요양병원 입원/선택기관 경고를 붙인다.
8. 의료급여 M3 승인과 M4 결과 저장을 구현한다.
9. 의료급여 M5 취소와 M6 결과 저장을 구현한다.
10. 비용계산과 SAM 청구에서 저장 필드를 참조하도록 연결한다.

## 14. 필수 테스트 케이스

| 케이스 | 검증 |
|---|---|
| 건강보험 직장가입자 | `InsurnerType=건강보험`, 증번호 저장 |
| 건강보험 지역가입자 | 국적구분 저장 |
| 건강보험 급여제한자 | `InsuranceLimitType`, 경고 메시지 |
| 출국자 | `IsExitCountry`, 건강보험_100 접수 안내 |
| 의료급여 1종 | `MedicalAid.Type=1`, 보장기관 저장 |
| 의료급여 2종 | `MedicalAid.Type=2`, 본인부담 코드 저장 |
| 의료급여 2종 장애 | 장애여부 `Y`이면 `MedicalAid.Type=8` |
| 선택기관 대상자 | `AppointHospital` 1~4차 저장, 시작일 저장 |
| 차상위 C/E/F | `GovernmentInjuryType`, `SpecialCase` 저장 |
| 희귀 H 지원대상 | `GovernmentInjuryType=H`, 지원금 계산 연결 |
| 암/중복암 산정특례 | 여러 `SpecialCase` 저장 |
| 중증치매/중증난치/잠복결핵 | 특례 기간/특정기호 저장 |
| 요양병원 입원자 | 입원 여부와 기관기호 저장, 접수 경고 |
| 비대면 대상자 | 산간벽지/장애인/장기요양 플래그 분리 |
| 의료급여 승인 | M3 요청 후 M4 `진료확인번호` 저장 |
| 의료급여 취소 | M5 요청 후 M6 취소 성공 및 잔액 복원 |

