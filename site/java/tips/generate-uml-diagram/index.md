@startuml
actor User as user
participant "PromiseService" as service
participant "PromiseRepository" as repository
participant "PromiseEventPublisher" as publisher
participant "PromiseRetryService" as retryService
participant "Kafka" as kafka

user -> service: processPromiseCommand()
activate service

service -> repository: save(promise)
activate repository
repository --> service: promise
deactivate repository

service -> publisher: publishEvent(promise)
activate publisher

publisher -> kafka: send(event)
activate kafka

alt Успешная отправка
    kafka --> publisher: success
    publisher --> service: success
    deactivate kafka
    deactivate publisher
    service --> user: success
else Ошибка отправки
    kafka --> publisher: error
    deactivate kafka
    publisher --> service: error
    deactivate publisher

    service -> retryService: retrySendEvent(promise)
    activate retryService

    loop Повторные попытки
        retryService -> publisher: publishEvent(promise)
        activate publisher
        publisher -> kafka: send(event)
        activate kafka

        alt Успешная отправка
            kafka --> publisher: success
            publisher --> retryService: success
            deactivate kafka
            deactivate publisher
            retryService --> service: success
            break
        else Ошибка отправки
            kafka --> publisher: error
            deactivate kafka
            publisher --> retryService: error
            deactivate publisher
        end
    end

    retryService --> service: retry failed
    deactivate retryService
    service --> user: error
end

deactivate service
@enduml
