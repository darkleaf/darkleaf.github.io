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
