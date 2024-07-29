```clojure
(def stop (ex-info "stop" {})

(try
  ((fn []
     (throw ex)))
  (catch Throwable ex
     (if (identical? ex stop)
       ...
       (throw ex))))
```

***

Протоколы - это не для пользоваателя (caller).
можно найти в гугл-юзер группе сообщение Стю.

Собственно ок делать еще один протокол.
Ну или интерфейс.

Вот clojure.lang.IAtom, а потом расширили через clojure.lang.IAtom2.
И мы пользуемся обретками над методами, а не самими методами

***

макросы в тестах

```clojure
(defmacro ...
`(-with-snapshot ~snapshot-name
                 (fn ~'snapshot-fail [msg#]
                    ;; чтобы можно было выключить fail-fast и собрать все ошибки
                     (t/do-report {:type    :fail
                                   :message (str ~snapshot-name "\r\n" msg#)})
```
так ошибка будет в коде теста, а не функи `-with-snapshot`
