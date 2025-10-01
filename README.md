---

# ⚙️ [내일배움캠프 언리얼엔진] GameState와 GameInstance로 게임 루프 완성하기

> ## **학습 키워드**
>
> *   `Game Loop`: 게임의 시작부터 종료까지 이어지는 전체적인 흐름과 규칙.
> *   `GameState` vs `GameMode`: 전역 '상태' 관리와 서버 '규칙' 관리의 역할 분담.
> *   `GameInstance`: 게임 세션 동안 파괴되지 않고 데이터를 유지하는 유일한 객체.
> *   `Level Streaming (OpenLevel)`: 레벨을 전환하는 방법과 데이터 초기화 문제.
> *   `TimerManager`: 지정된 시간 후에 특정 함수를 호출하는 기능.
> *   `UGameplayStatics`: 월드 내 액터를 찾거나 레벨을 여는 등 유용한 전역 함수 모음.

<br>

---

## **1. GameState를 이용한 게임 루프 구현하기**

---

지금까지 우리는 개별 아이템의 기능과 캐릭터의 기본 동작을 구현했습니다. 

이제는 "레벨 시작 → 아이템 스폰 → 시간 제한 → 코인 수집 → 다음 레벨로 전환 → 게임 오버"로 이어지는 **게임 루프(Game Loop)**를 설계하여 

하나의 완성된 게임 흐름을 만들 차례입니다.

### 1️⃣ **GameState vs GameMode: 역할의 선택**

게임의 전역 상태를 관리할 때 `GameState`와 `GameMode` 중 어떤 것을 사용해야 할까요?

*   **`GameMode`**

    게임의 핵심 **'규칙'**을 정의하며, 멀티플레이 환경에서는 **서버에만 존재**합니다.

    클라이언트가 직접 접근할 수 없으므로, 모든 플레이어가 알아야 할 정보(남은 시간, 현재 점수 등)를 관리하기에는 부적합합니다.

*   **`GameState`**

    게임의 **'상태'**를 저장하며, 서버와 모든 클라이언트가 **동일한 정보를 공유**합니다.

    따라서 남은 시간, 스폰된 아이템 수, 현재 레벨 등 모든 플레이어가 알아야 할 전역 데이터는 `GameState`에서 관리하는 것이 올바른 설계입니다.

이번 프로젝트에서는 레벨별 아이템 수, 남은 시간 등 공유해야 할 정보가 많으므로 **`GameState`**를 중심으로 게임 루프를 구현합니다.

### 2️⃣ **`AGameState` 기반의 게임 루프 구현**

`AGameState`를 상속받는 `AMyGameState` 클래스를 생성하고, 게임의 전체 흐름을 제어하는 변수와 함수를 추가합니다.

*   **`MyGameState.h` (헤더 파일)**
    ```cpp
    #pragma once
    
    #include "CoreMinimal.h"
    #include "GameFramework/GameState.h"
    #include "MyGameState.generated.h"
    
    UCLASS()
    class MYPROJECT_API AMyGameState : public AGameState
    {
    	GENERATED_BODY()
    
    public:
    	AMyGameState();
    
    	void AddScore(int32 Amount);
    	void OnCoinCollected();
    
    protected:
    	virtual void BeginPlay() override;
    
    	// 레벨별 정보
    	UPROPERTY(EditDefaultsOnly, Category="Level")
    	TArray<FName> LevelMapNames; // 진행할 레벨 맵 이름 목록
    
    	// 게임 상태 변수
    	int32 SpawnedCoinCount;
    	int32 CollectedCoinCount;
    	int32 CurrentLevelIndex;
    
    private:
    	// 레벨 시작, 종료 로직
    	void StartLevel();
    	void EndLevel();
    	void OnLevelTimeUp();
    	void OnGameOver();
    
    	FTimerHandle LevelTimerHandle;
    };
    ```
*   **`MyGameState.cpp` (소스 파일) - 핵심 로직**
    ```cpp
    #include "MyGameState.h"
    #include "Kismet/GameplayStatics.h"
    #include "MySpawnVolume.h" // 스폰 볼륨 클래스
    #include "MyCoinItem.h"    // 코인 타입 확인용
    
    void AMyGameState::BeginPlay()
    {
    	Super::BeginPlay();
    	StartLevel(); // 게임이 시작되면 첫 레벨을 시작합니다.
    }
    
    void AMyGameState::StartLevel()
    {
    	SpawnedCoinCount = 0;
    	CollectedCoinCount = 0;
    
    	// 1. 레벨에 있는 모든 스폰 볼륨을 찾아 40개의 아이템을 스폰합니다.
    	TArray<AActor*> FoundVolumes;
    	UGameplayStatics::GetAllActorsOfClass(this, AMySpawnVolume::StaticClass(), FoundVolumes);
    	if (FoundVolumes.Num() > 0)
    	{
    		AMySpawnVolume* SpawnVolume = Cast<AMySpawnVolume>(FoundVolumes[0]);
    		for (int32 i = 0; i < 40; ++i)
    		{
    			AActor* SpawnedActor = SpawnVolume->SpawnRandomItem();
    			// 2. 스폰된 아이템이 코인 계열이면 카운트를 올립니다.
    			if (SpawnedActor && SpawnedActor->IsA(AMyCoinItem::StaticClass()))
    			{
    				SpawnedCoinCount++;
    			}
    		}
    	}
    
    	// 3. 30초 후에 OnLevelTimeUp 함수를 호출하는 타이머를 설정합니다.
    	GetWorldTimerManager().SetTimer(LevelTimerHandle, this, &AMyGameState::OnLevelTimeUp, 30.0f, false);
    }
    
    void AMyGameState::OnCoinCollected()
    {
    	CollectedCoinCount++;
    	// 4. 모든 코인을 수집했다면, 즉시 레벨을 종료합니다.
    	if (SpawnedCoinCount > 0 && CollectedCoinCount >= SpawnedCoinCount)
    	{
    		EndLevel();
    	}
    }
    
    void AMyGameState::OnLevelTimeUp()
    {
        EndLevel(); // 시간이 다 되면 레벨을 종료합니다.
    }
    
    void AMyGameState::EndLevel()
    {
    	GetWorldTimerManager().ClearTimer(LevelTimerHandle); // 진행 중인 타이머를 해제합니다.
    	CurrentLevelIndex++;
    
    	// 5. 다음 레벨로 전환하거나 게임 오버를 처리합니다.
    	if (LevelMapNames.IsValidIndex(CurrentLevelIndex))
    	{
    		UGameplayStatics::OpenLevel(this, LevelMapNames[CurrentLevelIndex]);
    	}
    	else
    	{
    		OnGameOver();
    	}
    }
    
    void AMyGameState::OnGameOver()
    {
    	UE_LOG(LogTemp, Warning, TEXT("Game Over!"));
    	// 게임 오버 UI를 띄우거나 재시작 로직을 호출합니다.
    }
    ```
### 3️⃣ **코인 아이템과 GameState 연동**

코인 아이템을 획득했을 때, `GameState`에 알려주도록 `ActivateItem` 함수를 수정합니다.

*   **`CoinItem.cpp` (소스 파일)**
    ```cpp
    #include "MyGameState.h"
    
    void AMyCoinItem::ActivateItem(AActor* Activator)
    {
    	if (AMyGameState* MyGameState = GetWorld()->GetGameState<AMyGameState>())
    	{
    		MyGameState->AddScore(PointValue);
    		MyGameState->OnCoinCollected(); // 코인을 수집했다고 GameState에 알립니다.
    	}
    	DestroyItem();
    }
    ```

<br>

---

## **2. GameInstance로 레벨 간 데이터 유지하기**

---

`OpenLevel` 함수로 맵을 전환하면, 현재 레벨의 모든 액터(GameState 포함)는 파괴되고 새로운 맵의 액터들이 새로 생성됩니다. 

이 때문에 **다음 레벨로 넘어가면 점수가 초기화되는 문제**가 발생합니다.

이 문제를 해결하기 위해 **GameInstance**를 사용합니다.

### 1️⃣ **GameInstance란?**

**GameInstance**는 게임이 실행될 때 단 하나만 생성되어, 

**게임이 완전히 종료될 때까지 파괴되지 않고 살아있는 특별한 객체**입니다. 

덕분에 여러 레벨에 걸쳐 유지되어야 할 데이터(누적 점수, 플레이어 설정 등)를 저장하기에 가장 적합한 장소입니다.

### 2️⃣ **`UMyGameInstance` 생성 및 데이터 저장**

`UGameInstance`를 상속받는 `UMyGameInstance` 클래스를 생성하고, 레벨 전환 시에도 유지할 데이터를 선언합니다.

*   **`MyGameInstance.h` (헤더 파일)**
    ```cpp
    #pragma once
    
    #include "CoreMinimal.h"
    #include "Engine/GameInstance.h"
    #include "MyGameInstance.generated.h"
    
    UCLASS()
    class MYPROJECT_API UMyGameInstance : public UGameInstance
    {
    	GENERATED_BODY()
    
    public:
    	UMyGameInstance();
    
    	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GameData")
    	int32 TotalScore; // 게임 전체 누적 점수
    
    	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GameData")
    	int32 CurrentLevelIndex; // 현재 진행 중인 레벨 인덱스
    
    	UFUNCTION(BlueprintCallable)
    	void AddToTotalScore(int32 Amount);
    };
    ```

### 3️⃣ **GameState와 GameInstance 연동**

이제 레벨이 시작될 때 `GameInstance`에서 데이터를 가져와 초기화하고, 레벨이 끝날 때 현재 상태를 `GameInstance`에 다시 저장합니다.

*   **`MyGameState.cpp` (소스 파일)**
    ```cpp
    #include "MyGameInstance.h"
    
    void AMyGameState::BeginPlay()
    {
    	Super::BeginPlay();
    
        // 게임 인스턴스에서 현재 레벨 인덱스를 가져와 설정
    	if (UMyGameInstance* GameInstance = GetGameInstance<UMyGameInstance>())
    	{
    		CurrentLevelIndex = GameInstance->CurrentLevelIndex;
    	}
    
    	StartLevel();
    }
    
    void AMyGameState::EndLevel()
    {
        // ...
        if (UMyGameInstance* GameInstance = GetGameInstance<UMyGameInstance>())
        {
            // 현재 레벨에서 얻은 점수를 GameInstance의 총점에 더함
            GameInstance->AddToTotalScore(CurrentScore); 
            // 다음 레벨 인덱스를 GameInstance에 저장
            GameInstance->CurrentLevelIndex = CurrentLevelIndex + 1;
        }
        // ... 레벨 전환 또는 게임 오버 처리
    }
    ```

마지막으로 **Project Settings > Maps & Modes > Game Instance Class**에서 

우리가 만든 `UMyGameInstance`(또는 이를 기반으로 한 블루프린트)를 지정해주면, 

레벨을 넘나들어도 점수가 초기화되지 않고 계속 누적되는 것을 확인할 수 있습니다. 

즉 게임 루프가 완성되었습니다.

---

#언리얼엔진 #게임루프 #GameState #GameInstance #CPP #내일배움캠프
