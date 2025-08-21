# bevy_godot4
# ![logo](logo_long.png)

<!-- > **NOTICE**: This crate is currrently unmaintained, and due to changes in gdext's api it is pinned to an old version of gdext and only works with Godot 4.0 -->

Bevy의 ECS 설계 능력을 Godot 4의 성숙한 엔진 기능에 접목하세요.

핵심적으로, 이 크레이트는 Godot 프로젝트에 자동로드로 추가하는 Bevy `App`을 저장하는 Godot 노드일 뿐입니다. 하지만 이 라이브러리는 Bevy 프레임워크 내에서 Godot 노드와 함께 작동할 수 있는 유틸리티도 제공합니다.

이 크레이트의 아키텍처는 Godot 3 및 GDNative와 함께 작동하는 유사한 크레이트인 [bevy_godot](https://github.com/rand0m-cloud/bevy_godot)을 기반으로 합니다.

## 설정

1. [GDExtension 시작하기](https://godot-rust.github.io/book/intro/index.html)에 설명된 단계를 따르세요.

2. 이 라이브러리를 의존성으로 추가하세요 (GDExtension godot 크레이트와 함께):
```toml
[dependencies]
bevy = { version = "0.16", default-features = false, features = [
    "bevy_asset",
    "bevy_state",
] }
bevy_godot4 = { git = "https://github.com/jrockett6/bevy_godot4", branch = "main" }
godot = "0.2.4"
```

> **_참고:_** 물론 `bevy`의 다른 기능들을 활성화할 수 있습니다. 위 예제에서는 컴파일 시간과 빌드 결과물의 크기를 최소화하기 위해 기능 세트를 최소화했습니다.

3. `&mut App`을 인자로 받아 bevy 앱을 빌드하는 함수를 만들고, `#[bevy_app]`으로 어노테이션을 추가하세요:
```rust
#[bevy_app]
fn build_app(app: &mut App) {
    app.add_system(my_system)
}
```

4. 프로젝트를 `cargo build`하고, `.gdextension` 파일을 통해 Godot이 dll을 찾을 수 있도록 하세요. 이제 Godot 에디터에서 `BevyApp` 노드를 사용할 수 있습니다 (에디터에서 프로젝트를 새로고침해야 할 수도 있습니다).

5. 이 `BevyApp` 노드를 새로운 씬 `bevy_app_singleton.tscn`의 루트로 추가하고, Godot 프로젝트 설정에서 `BevyAppSingleton`이라는 이름의 Godot 자동로드로 씬을 추가하세요.

## 버전 호환성 매트릭스

| bevy_godot4 | Bevy | Godot-Rust | Godot |
|-------------|------|------------|-------|
| 0.2.x       | 0.15 | 0.2.4      | 4.4.x |
| 0.3.x       | 0.16 | 0.2.4      | 4.4.x |


## 기능

### 컴포넌트로서의 Godot 노드
`ErasedGd`는 Godot 노드 인스턴스 ID를 보유하는 Bevy 컴포넌트입니다. 시스템 내에서 이를 `Query`하고 `get::<T>()` 또는 `try_get::<T>()`을 사용하여 노드를 가져올 수 있습니다.
```rust
fn set_positions(mut erased_gds: Query<&mut ErasedGd>) {
    for mut node in erased_gds.iter_mut() {
        if let Some(node2D) = node.try_get::<Node2D>() {
            node2D.set_position(Vector2::ZERO)
        }
    }
}
```

### Bevy 리소스로서의 Godot 리소스
마찬가지로, `ErasedGdResource`는 `Send` & `Sync`이며 `RefCounted` Godot `Resource` 타입을 보유할 수 있습니다.
```rust
#[derive(Resource)]
pub struct GodotResources {
    pub my_packed_scene: ErasedGdResource,
}

impl Default for GodotResources {
    // load godot resource with e.g. the ResourceLoader singleton.
}

app.init_resource::<GodotResources>();
```

### Erased PackedScene 리소스에서 Godot 씬 스폰
`GodotScene`은 `PackedScene` 인스턴스화, 씬 트리에 추가, 그리고 Bevy 시스템에서 작업할 해당 `ErasedGd` 추가를 처리합니다.
```rust
fn spawn_scene(
    godot_resources: Res<GodotResources>,
    mut commands: Commands,
) {
    commands.spawn(GodotScene::from_resource(godot_resources.my_packed_scene.clone()));
}
```

### _process 또는 _physics_process 업데이트 루프를 위한 시스템 스케줄링
`as_visual_system()` 또는 `as_physics_system()`은 Bevy 시스템이 원하는 Godot 업데이트 루프에서 실행되도록 보장합니다.
``` rust
app.add_system(set_positions.as_physics_system())
```

### Bevy 시스템이 메인 스레드에서 실행되도록 보장
`SceneTreeRef`는 Bevy 시스템이 메인 스레드에서 스케줄링되도록 보장하는 `NonSend` 시스템 파라미터입니다.
```rust
fn my_main_thread_system(
    ...,
    _scene_tree: SceneTreeRef,
) {
    // non-threadsafe code, e.g. move_and_slide()
}
```

*더 많은 정보는 examples 폴더를 확인하세요.*
