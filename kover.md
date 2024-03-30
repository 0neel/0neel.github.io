# Настройка Kover в многомодульном Android-проекте

> 💡 [Kover](https://github.com/Kotlin/kotlinx-kover) - это утилита (и Gradle-плагин) от JetBrains для сбора данных по покрытию тестами (test coverage).

[Официальные доки](https://kotlin.github.io/kotlinx-kover/gradle-plugin/) не объясняют прямо как настроить проект, поэтому пришлось разбираться самостоятельно.

Надеюсь этот гайд поможет вам пройти мой путь быстрее.

Буду рад ответить на любые вопросы в:

- [Telegram](https://t.me/oneel12)
- [Twitter](https://twitter.com/0neel)

> 💡 Гайд актуален для версии [0.7.6](https://github.com/Kotlin/kotlinx-kover/releases/tag/v0.7.6). DSL плагина постоянно меняется, поэтому с новой версией что-то обязательно сломается.

## TL;DR

<details>
<summary>Хочется просто посмотреть полный конфиг? Пожалуйста!</summary>

```groovy
plugins {
  id "org.jetbrains.kotlinx.kover" version "0.7.6" apply false
}

subprojects {
  apply plugin: "org.jetbrains.kotlinx.kover"

  kover {
    useJacoco()
  }

  koverReport {
    filters {
      excludes {
        classes(
            // Android
            '**/R.class',
            '**/R$*.class',
            '**/BuildConfig.*',
            '**/Manifest*.*',
            // Several "$" in class name (not supported by JaCoCo)
            '**/*$$*',
            '**/*$Lambda$*.*',
            '**/*$inlined$*.*',
            // Dagger
            '**/*Dagger*.*',
            '**/*MembersInjector*.*',
            '**/*_Provide*Factory*.*',
            '**/*_Factory*.*',
            // Realm
            '**/io/realm/*'
        )
      }
    }
  }
  
  plugins.withType(com.android.build.gradle.api.AndroidBasePlugin) {
    configurations.implementation {
      dependencies.withType(ProjectDependency) {
        project.dependencies.kover dependencyProject
      }
    }

    configurations.api {
      dependencies.withType(ProjectDependency) {
        project.dependencies.kover dependencyProject
      }
    }
  }
}
```

</details>
    

## Почему именно Kover?

Я долго откладывал использование Kover, потому что у авторов очень творческий подход к дизайну плагина - синтаксис настройки меняется с каждой минорной версией. Понимаю, релиз 1.0 ещё не вышел, можно менять что угодно. Правда, в таком состоянии проект находится уже 3 года. Сложно говорить о production-ready с таким отношением (привет ранним версиям Swift!).

Поэтому, пару лет назад, когда надо было настроить покрытие в проекте, я выбрал замечательный  https://github.com/NeoTech-Software/Android-Root-Coverage-Plugin. Он максимально прост в настройке и заточен под Android-проекты. К сожалению у автора возникли сложности с поддержкой AGP 8 (на момент написания статьи ещё нет релиза, который поддерживает его). А переход на AGP 8 - это обязательное условие для targetSdk 34. Одно потянулось за другое. Дальше пользоваться rootCoverage было невозможно. Прощай, милый друг.

Kover выделяется среди остальных вариантов тем, что поддерживается JetBrains, а значит можно расчитывать на быстрое решние проблем совместимости с AGP. Сами же данные по покрытию, я надеюсь, будут максимально точными - кто может точнее собирать данные по Kotlin, если не его авторы?

После перехода процент покрытия слегка поменялся, даже с учетом того, что я включил JaCoCo в качестве движка для Kover (rootCoverage тоже работает на JaCoCo). Это не проблема - в покрытии не так важна абсолютная цифра, как динамика.

## Пошаговая настройка

Весь код нужно добавлять в корневой `build.gradle` проекта.

- Добавляем зависимость на плагин
    
    ```groovy
    plugins {
      id "org.jetbrains.kotlinx.kover" version "0.7.6" apply false
    }
    ```
    
    Указываем `apply false` потому что в корневом проекте плагин не используется. Настройка будет происходить для каждого модуля отдельно.
    
- Подключаем плагин в каждый из подпроектов (модулей с кодом)
    
    ```groovy
    subprojects {
      apply plugin: "org.jetbrains.kotlinx.kover"
    }
    ```
    
- (Опционально) Настраиваем Kover
    - Указываем JaCoCo в качестве инструмента для сбора покрытия.
        
        ```groovy
        subprojects {
          // ...
          
          kover {
            useJacoco()
          }
        }
        ```
        
        Мы в проекте используем SonarCloud для визуализации покрытия, а он не умеет работать со встроенным в Kover инструментом - к счастью, можно переключиться на JaCoCo.
        
    - Настраиваем файлы, которые не нужно учитывать при сборе покрытия
        
        ```groovy
        subprojects {
          // ...
          
          koverReport {
            filters {
              excludes {
                classes(
                    // Android
                    '**/R.class',
                    '**/R$*.class',
                    '**/BuildConfig.*',
                    '**/Manifest*.*',
                    // Several "$" in class name (not supported by JaCoCo)
                    '**/*$$*',
                    '**/*$Lambda$*.*',
                    '**/*$inlined$*.*',
                    // Dagger
                    '**/*Dagger*.*',
                    '**/*MembersInjector*.*',
                    '**/*_Provide*Factory*.*',
                    '**/*_Factory*.*',
                    // Realm
                    '**/io/realm/*'
                )
              }
            }
          }
        }
        ```
        
        Если не используете Dagger и/или Realm, то для чистоты соответствующие строки стоит удалить.
        
- Настраиваем автоматическую генерацию единого репорта для всего проекта.
    
    ```groovy
    subprojects {
      // ...
      
      plugins.withType(com.android.build.gradle.api.AndroidBasePlugin) {
        configurations.implementation {
          dependencies.withType(ProjectDependency) {
            project.dependencies.kover dependencyProject
          }
        }
    
        configurations.api {
          dependencies.withType(ProjectDependency) {
            project.dependencies.kover dependencyProject
          }
        }
      }
    }
    ```
    
    <details>
    <summary>Что тут происходит</summary>

    Чтобы сгенерировать единый репорт, в официальной доке предлагают во все модули проекта добавлять [специальный тип зависимостей](https://github.com/Kotlin/kotlinx-kover/tree/v0.7.6?tab=readme-ov-file#to-create-report-combining-coverage-info-from-different-gradle-projects) - kover.
    
    ```groovy
    dependencies {
      implementation(project(":another:project")) // обычная зависимость
      kover(project(":another:project")) // а такую надо добавить
    }
    ```
    
    Получается, надо продублировать все зависимости по всему проекту.
    Это слишком много ручной работы.
    
    Поэтому мы сделаем это автоматически!
    
    - Сначала мы ищем все Android-модули
        
        ```groovy
        plugins.withType(com.android.build.gradle.api.AndroidBasePlugin) {
        ```
        
    - Настраиваем его implementation и api-конфигурации
        
        ```groovy
        configurations.implementation {
        // ...
        configurations.api {
        ```
        
    - Для каждой из них находим зависимости от других модулей
        
        ```groovy
        dependencies.withType(ProjectDependency) {
        ```
        
    - И, наконец, для каждой такой зависимости добавляем её kover-копию
        
        ```groovy
        project.dependencies.kover dependencyProject
        ```
    
    </details>
            
- Готово! Пора сделать Gradle Sync и собрать покрытие.
    
    Например, для репорта в формате XML по debug-варианту приложения нужно запустить Gradle-таск `koverXmlReportDebug`.
    
    Репорт будет сгенерирован в файле `./app/build/reports/kover/reportDebug.xml`.
    

## Бонус: связываем Kover с SonarQube/SonarCloud

Для отображения данных от Kover в плагине Sonar нужно настроить свойство `sonar.coverage.jacoco.xmlReportPaths`.

Это можно сделать так:

```groovy
sonar {
  properties {
    property 'sonar.coverage.jacoco.xmlReportPaths', "${rootProject.rootDir}/**/build/reports/kover/report*.xml"
  }
}
```

Теперь любой репорт, который сгенерирует Kover, будет отправлен в Sonar.

Хочешь полный гайд по настройке плагина Sonar? Напиши мне!
