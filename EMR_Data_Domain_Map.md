# EMR Data Domain Map

대상: `Data/UBerSoft.EMR.Data`

이 문서는 코드 구조 설명이 아니라, 현재 Data 계층에 들어 있는 주요 업무 도메인과 규칙의 위치를 빠르게 잡기 위한 1차 지도이다.

## 핵심 도메인 축

### 1. 진료비/본인부담 계산

주요 파일:

- `Services/Computes/Concrete/AmountService.cs`
- `Services/Computes/Concrete/PatientCostServiceManager.cs`
- `Services/Computes/Concrete/PatientCostServiceBase.cs`
- `Services/Computes/Concrete/PatientCostService_Insurance*.cs`
- `Services/Computes/Concrete/PatientCostService_MedicalAid.cs`
- `Services/Computes/Concrete/PatientCostService_NoInsurance.cs`
- `Services/Computes/Concrete/PatientCosts/*.cs`
- `Adaptors/Concrete/PatientCosts/*.cs`

업무 의미:

- 처방 라인의 금액을 계산하고, 청구 명세서 금액 항목으로 분리한다.
- 보험자 유형, 환자 나이, 차상위/산정특례/의료급여/공상 여부에 따라 본인부담금을 산정한다.
- 건강보험 기본 계산 후 산정특례 계산 결과가 있으면 더 낮은 본인부담 결과를 선택한다.
- 요양급여비용총액1, 요양급여비용총액2, 본인일부부담금, 청구액, 장애인의료비, 지원금, 100/100 및 100/100 미만 항목을 계산한다.

상세 문서:

- `Docs/Patient_Cost_Domain.md`

### 2. 보험청구/명세서/SAM

주요 파일:

- `Services/InsuranceCharges/Concrete/BillService.cs`
- `Services/InsuranceCharges/Concrete/InsuranceChargeValidater.cs`
- `Services/InsuranceCharges/Concrete/SpecificDetailService.cs`
- `Services/InsuranceCharges/Concrete/SAMs/Sends/*.cs`
- `Services/InsuranceCharges/Concrete/SAMs/Receives/*.cs`
- `Repositories/InsuranceCharges/*.cs`
- `Controllers/InsuraceCharges/Concreate/*.cs`

업무 의미:

- 월 단위 또는 지정 단위의 진료 목록을 건강보험 청구서와 의료급여 청구서로 분리 생성한다.
- 개별 진료를 명세서로 변환하고, 동일 수진자/동일 내원일/동일 주상병 진료는 조건부로 명세서를 병합한다.
- 명세서 병합 시 상병, 처방, 처방전, 특정내역을 재배열하고 금액을 재계산한다.
- 추가청구는 기존 청구 명세서와 비교해 이미 청구된 처방/처방전/금액을 차감한다.
- SAM 파일은 청구서 버전별 writer를 선택해 `.GHP` 파일로 생성하고, HIRA DDMD 암호화/점검/송신 프로그램과 연동한다.
- 청구 전 검증은 수진자, 상병, 처방, 금액, 의료급여 확인번호, 중복청구 위험을 검사한다.

핵심 규칙 후보:

- 건강보험/의료급여 별도 청구서 생성.
- 명세서 병합 제외 조건: MT001 상해외인 특정내역이 있는 경우.
- 의료급여 또는 차상위성 구분에서는 병합 명세서에 MT040 본인부담 발생횟수 추가.
- 데모/개발/요양기관 코드 `0000` 시작 병원은 상시점검 파일로 생성.

### 3. 진료 작성: 상병/처방 변환과 묶음 병합

주요 파일:

- `Services/Concrete/DiseaseToTreatmentService.cs`
- `Services/Concrete/PrescribeToTreatmentService.cs`
- `Services/Concrete/TreatmentUnionService.cs`
- `Repositories/Treatments/TreatmentRepository.cs`
- `Controllers/Treatements/Concreate/TreatmentController.cs`

업무 의미:

- 마스터 상병을 진료 상병으로 변환하면서 진료과, 진료결과, 면허번호를 기본 세팅한다.
- 주상병에는 환자 보험의 산정특례 정보를 기준으로 특정기호를 자동 부여한다.
- 처방 마스터를 진료 처방으로 변환하면서 보험/비급여 구분, 항/목, 원내외, 용법, 면허번호, 위탁검사, 의약분업 예외코드를 세팅한다.
- 묶음/반복 처방을 기존 진료에 합칠 때 주상병 중복, 기본진찰료 중복, 급여 가능 여부, 위탁검사 날짜, Bundle parent-child 관계를 정리한다.

핵심 규칙 후보:

- 일반 환자는 처방이 비급여로 전환된다.
- 급여 환자라도 해당 진료일에 급여 적용 불가인 처방은 비급여로 변경되거나 경고 대상이 된다.
- 원내 주사제는 의약분업 예외코드 `52`가 기본 설정된다.
- 기존 진료에 주상병이 있으면 추가 상병의 주상병 여부와 진료결과/면허번호를 조정한다.

### 4. 기준 코드/마스터

상세 문서: `Base_Code_Domain.md`

주요 파일:

- `Contexts/CodeContext.cs`
- `Controllers/Codes/Concreate/*.cs`
- `Repositories/Codes/*.cs`
- `Services/Codes/Concrete/MasterToUserService.cs`

업무 의미:

- 상병, 약품, 수가, 치료재료, 행위료, 특정기호, 용법, 병원, 진료실, 공휴일, 서식문구, 묶음 등 EMR 운영 기준 데이터를 제공한다.
- 코드 컨텍스트는 화면에서 쓰는 공통 코드 목록을 조합한다.
- 마스터 코드와 병원/사용자 코드 사이의 복사, 검색, 등록 흐름이 존재한다.

핵심 규칙 후보:

- 코드 검색 계층은 `Searchable*`, 등록/수정 계층은 `Manage*`, 마스터 코드 계층은 `Master*` 베이스로 나뉜다.
- 처방 검색은 약품, 수가, 치료재료, 원료약, 성분명, 묶음 등 처방 유형별 controller로 분기된다.

### 5. 접수/수납/환불/마감/통계

주요 파일:

- `Controllers/ReceptionDesk/Concreate/*.cs`
- `Repositories/ReceptionDesk/*.cs`
- `Controllers/State/Concreate/*.cs`
- `Repositories/State/*.cs`

업무 의미:

- 대기 진료, 대기 수납, 수납, 진료-수납 연결, 환불 대상 조회를 담당한다.
- 일마감, 월마감, 수납 현황, 약품/행위/치료재료 사용 현황, 비급여 보고, 소득공제, 위탁검사 통계를 조회한다.

현재 파악 수준:

- 이 영역은 Data 프로젝트 내부에서는 주로 API 호출과 controller/repository 전달 구조가 중심이다.
- 상세한 수납 상태 전이와 마감 기준은 서버 API 또는 ViewModel/Model 쪽에 더 많이 있을 가능성이 있다.

### 6. 마약류/NIMS

주요 파일:

- `Controllers/Narcotics/*.cs`
- `Repositories/Narcotics/*.cs`
- `Services/Narcotics/BarcodeToReportService.cs`
- `Services/Concrete/BarcodeService.cs`
- `Services/Concrete/RFIDService.cs`

업무 의미:

- 마약류 저장소, 구입, 조제/처방, 보고, 수신 결과, 도매상, 제품 조회 흐름을 담당한다.
- 바코드/RFID를 통해 제품 식별 정보를 보고 데이터로 변환한다.

현재 파악 수준:

- NIMS 도메인은 controller/repository가 넓고, 일부 바코드 변환 서비스에 실제 규칙이 있다.
- 다음 분석 단계에서 별도 문서로 분리하는 것이 좋다.

### 7. 제증명/리포트/출력

주요 파일:

- `Controllers/Reports/Concreate/*.cs`
- `Repositories/Reports/*.cs`
- `Services/OpenXmls/ViewModelToSheetService.cs`
- `Services/OrderReports/Concrete/*.cs`
- `Services/QrCodes/Concrete/QrCodeService_EDB.cs`

업무 의미:

- 진단서, 진료확인서, 입퇴원확인서, 처방전, 영수증, 진료비 세부산정내역, 의뢰서, 수술확인서 등 문서 데이터를 조회/생성한다.
- 출력, 엑셀 변환, QR 코드 생성, 오더 출력 서비스가 포함된다.

현재 파악 수준:

- 문서별 데이터 조회는 repository/API 중심이다.
- 실제 서식 레이아웃은 다른 프로젝트 또는 리포트 템플릿에 있을 가능성이 높다.

## 우선 분석 순서 제안

1. 진료비/본인부담 계산
2. 보험청구/명세서/SAM
3. 진료 작성: 상병/처방/묶음 병합
4. 마약류/NIMS
5. 접수/수납/마감
6. 기준 코드/마스터
7. 제증명/리포트

## 분석 시 주의점

- 이 프로젝트는 Data 계층이지만 단순 데이터 접근만 하지 않는다. 본인부담, 청구 명세서, 특정기호, 처방 변환 같은 실제 업무 규칙이 서비스에 들어 있다.
- 일부 도메인 규칙은 `UBerSoft.EMR.Data.Models`, `UBerSoft.EMR.Data.ViewModels`, `UBerSoft.EMR.Core`, 서버 API에도 나뉘어 있을 가능성이 높다.
- 현재 문서는 `Data/UBerSoft.EMR.Data` 내부에서 확인 가능한 규칙만 1차 정리했다.
