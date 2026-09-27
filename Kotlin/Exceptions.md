# Kotlin — checked exceptions

### Почему в Kotlin нет checked exceptions
- Checked exceptions нарушают принцип подстановки Лисков
- Приводят к шаблонному коду и пустым catch-блокам
- Не дают реальных гарантий — их всё равно игнорируют
- Большие API становятся неудобными (каждый метод throws IOException)

---
