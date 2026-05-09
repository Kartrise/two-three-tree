# Організація пошукових індексів на основі 2-3 дерев

## Постановка задачі

Алгоритм розв'язує задачу організації **пошукового індексу** - структури, яка дозволяє ефективно зберігати, шукати, вставляти та видаляти ключі (наприклад, слова або числові ідентифікатори документів). Для реалізації такого індексу застосовується 2-3 дерево - збалансована структура даних, яка гарантує однакову висоту для всіх шляхів від кореня до листів.

**ВХІД**: Послідовність цілочисельних ключів та набір операцій над ними:
- вставка ключа до індексу
- пошук ключа в індексі
- видалення ключа з індексу

**ВИХІД**:
- результат пошуку: вузол дерева та кількість входжень ключа (або повідомлення про відсутність)
- оновлена структура індексу після вставки / видалення
- відсортована послідовність усіх ключів індексу (in-order обхід)

Якщо один і той самий ключ зустрічається кілька разів - це відображається у лічильнику входжень, а не у структурі дерева. Фізичне видалення відбувається лише тоді, коли лічильник досягає нуля.

---

## Обґрунтування вибору структури даних

Для організації пошукового індексу обрано **2-3 дерево**, розроблене Дж. Хопкрофтом у 1970 р.

Ключова перевага перед звичайним бінарним деревом пошуку (BST) - **гарантована збалансованість**. У BST у найгіршому випадку (посортована вхідна послідовність) дерево виродиться у список і пошук займе O(N) часу. У 2-3 дереві це неможливо: балансування відбувається автоматично через механізм розщеплення вузлів при вставці та злиття/перерозподілу при видаленні.

Дерево підтримує вузли двох типів:
- **2-вузол** - містить один ключ K і має двох нащадків: лівий (ключі < K) і правий (ключі > K)
- **3-вузол** - містить два впорядковані ключі K₁ < K₂ і має трьох нащадків: лівий (ключі < K₁), середній (K₁ < ключі < K₂) і правий (ключі > K₂)

Усі листя дерева завжди знаходяться на **одному рівні**, що є головною інваріантою структури.

Для обробки дублікатів (один і той самий термін зустрічається в кількох документах) використовується окремий словник `counts` із лічильником входжень. Це не порушує структуру дерева і відповідає реальній логіці пошукового індексу.

---

## Код програми

```python
UNDERFLOW = object()

class Node:
    def __init__(self, key):
        self.key1  = key
        self.key2  = None
        self.left  = None
        self.mid   = None
        self.right = None

    def is_leaf(self):
        return self.left is None

    def is_3node(self):
        return self.key2 is not None

    def __repr__(self):
        return f"[{self.key1},{self.key2}]" if self.is_3node() else f"[{self.key1}]"

class TwoThreeTree:

    def __init__(self):
        self.root   = None
        self.counts = {}

    def insert(self, key):
        if key in self.counts:
            self.counts[key] += 1
            return
        self.counts[key] = 1

        if self.root is None:
            self.root = Node(key)
            return

        result = self._insert(self.root, key)

        if result is not None:
            promo, new_right   = result
            new_root           = Node(promo)
            new_root.left      = self.root
            new_root.mid       = new_right
            self.root          = new_root

    def _insert(self, node, key):
        if node.is_leaf():
            return self._insert_to_leaf(node, key)

        if key < node.key1:
            result, direction = self._insert(node.left, key), 'left'
        elif (not node.is_3node()) or key < node.key2:
            result, direction = self._insert(node.mid, key), 'mid'
        else:
            result, direction = self._insert(node.right, key), 'right'

        if result is None:
            return None
        promo, new_right = result
        return self._push_up(node, promo, new_right, direction)

    def _insert_to_leaf(self, node, key):
        if not node.is_3node():
            if key < node.key1:
                node.key2, node.key1 = node.key1, key
            else:
                node.key2 = key
            return None
        return self._split_leaf(node, key)

    def _split_leaf(self, node, key):
        k_small, k_mid, k_large = sorted([node.key1, node.key2, key])
        node.key1 = k_small
        node.key2 = None
        return (k_mid, Node(k_large))

    def _push_up(self, node, promo, new_right, direction):
        if not node.is_3node():
            if direction == 'left':
                node.key2  = node.key1
                node.key1  = promo
                node.right = node.mid
                node.mid   = new_right
            else:
                node.key2  = promo
                node.right = new_right
            return None
        return self._split_internal(node, promo, new_right, direction)

    def _split_internal(self, node, promo, new_right, direction):
        if direction == 'left':
            c = [node.left, new_right, node.mid, node.right]
        elif direction == 'mid':
            c = [node.left, node.mid, new_right, node.right]
        else:
            c = [node.left, node.mid, node.right, new_right]

        k_small, k_mid, k_large = sorted([node.key1, node.key2, promo])

        node.key1 = k_small; node.key2 = None
        node.left = c[0];    node.mid  = c[1]; node.right = None

        rn = Node(k_large)
        rn.left = c[2]; rn.mid = c[3]
        return (k_mid, rn)

    def delete(self, key):
        if key not in self.counts:
            return False

        self.counts[key] -= 1

        if self.counts[key] > 0:
            return True

        del self.counts[key]

        result = self._delete(self.root, key)

        if self.root is not None and self.root.key1 is None:
            self.root = self.root.left

        return result is not False

    def _delete(self, node, key):
        if node.is_leaf():
            return self._delete_from_leaf(node, key)
        if key == node.key1:
            succ = self._min_key(node.mid)
            node.key1 = succ
            result, direction = self._delete(node.mid, succ), 'mid'
        elif node.is_3node() and key == node.key2:
            succ = self._min_key(node.right)
            node.key2 = succ
            result, direction = self._delete(node.right, succ), 'right'
        elif key < node.key1:
            result, direction = self._delete(node.left, key), 'left'
        elif (not node.is_3node()) or key < node.key2:
            result, direction = self._delete(node.mid, key), 'mid'
        else:
            result, direction = self._delete(node.right, key), 'right'

        if result is False:
            return False
        if result is UNDERFLOW:
            return self._fix_underflow(node, direction)
        return True

    def _delete_from_leaf(self, node, key):
        if key == node.key1:
            if node.is_3node():
                node.key1 = node.key2
                node.key2 = None
                return True
            else:
                node.key1 = None
                return UNDERFLOW
        elif node.is_3node() and key == node.key2:
            node.key2 = None
            return True
        return False

    def _min_key(self, node):
        while not node.is_leaf():
            node = node.left
        return node.key1

    def _fix_underflow(self, parent, uf_dir):
        uf      = self._get_child(parent, uf_dir)
        is_leaf = (uf.left is None)

        if parent.is_3node():
            return self._fix_3node_parent(parent, uf_dir, uf, is_leaf)
        else:
            return self._fix_2node_parent(parent, uf_dir, uf, is_leaf)

    def _get_child(self, node, direction):
        if direction == 'left':  return node.left
        elif direction == 'mid': return node.mid
        else:                    return node.right

    def _fix_2node_parent(self, parent, uf_dir, uf, is_leaf):
        if uf_dir == 'left':
            sib = parent.mid
            sep = parent.key1

            if sib.is_3node():
                uf.key1      = sep
                parent.key1  = sib.key1
                sib.key1     = sib.key2
                sib.key2     = None
                if not is_leaf:
                    uf.mid   = sib.left
                    sib.left = sib.mid
                    sib.mid  = sib.right
                    sib.right = None
                return True
            else:
                sib.key2 = sib.key1
                sib.key1 = sep
                if not is_leaf:
                    sib.right = sib.mid
                    sib.mid   = sib.left
                    sib.left  = uf.left
                parent.left  = sib
                parent.key1  = None
                return UNDERFLOW

        else:
            sib = parent.left
            sep = parent.key1

            if sib.is_3node():
                uf.key1      = sep
                parent.key1  = sib.key2
                sib.key2     = None
                if not is_leaf:
                    uf.mid   = uf.left
                    uf.left  = sib.right
                    sib.right = None
                return True
            else:
                sib.key2 = sep
                if not is_leaf:
                    sib.right = uf.left
                parent.key1 = None
                return UNDERFLOW

    def _fix_3node_parent(self, parent, uf_dir, uf, is_leaf):
        if uf_dir == 'left':
            sib = parent.mid
            sep = parent.key1

            if sib.is_3node():
                uf.key1      = sep
                parent.key1  = sib.key1
                sib.key1     = sib.key2
                sib.key2     = None
                if not is_leaf:
                    uf.mid   = sib.left
                    sib.left = sib.mid
                    sib.mid  = sib.right
                    sib.right = None
            else:
                sib.key2 = sib.key1
                sib.key1 = sep
                if not is_leaf:
                    sib.right = sib.mid
                    sib.mid   = sib.left
                    sib.left  = uf.left
                parent.left  = sib
                parent.key1  = parent.key2
                parent.key2  = None
                parent.mid   = parent.right
                parent.right = None

        elif uf_dir == 'mid':
            r_sib = parent.right
            l_sib = parent.left

            if r_sib.is_3node():
                uf.key1       = parent.key2
                parent.key2   = r_sib.key1
                r_sib.key1    = r_sib.key2
                r_sib.key2    = None
                if not is_leaf:
                    uf.mid    = r_sib.left
                    r_sib.left = r_sib.mid
                    r_sib.mid = r_sib.right
                    r_sib.right = None
            elif l_sib.is_3node():
                uf.key1       = parent.key1
                parent.key1   = l_sib.key2
                l_sib.key2    = None
                if not is_leaf:
                    uf.mid   = uf.left
                    uf.left  = l_sib.right
                    l_sib.right = None
            else:
                r_sib.key2 = r_sib.key1
                r_sib.key1 = parent.key2
                if not is_leaf:
                    r_sib.right = r_sib.mid
                    r_sib.mid   = r_sib.left
                    r_sib.left  = uf.left
                parent.mid   = r_sib
                parent.key2  = None
                parent.right = None

        else:
            sib = parent.mid
            sep = parent.key2

            if sib.is_3node():
                uf.key1      = sep
                parent.key2  = sib.key2
                sib.key2     = None
                if not is_leaf:
                    uf.mid   = uf.left
                    uf.left  = sib.right
                    sib.right = None
            else:
                sib.key2 = sep
                if not is_leaf:
                    sib.right = uf.left
                parent.key2  = None
                parent.right = None

        return True

    def search(self, key):
        node = self._search(self.root, key)
        count = self.counts.get(key, 0)
        return (node, count)

    def _search(self, node, key):
        if node is None:
            return None
        if key == node.key1:
            return node
        if node.is_3node() and key == node.key2:
            return node
        if node.is_leaf():
            return None
        if key < node.key1:
            return self._search(node.left, key)
        elif (not node.is_3node()) or key < node.key2:
            return self._search(node.mid, key)
        else:
            return self._search(node.right, key)

    def inorder(self):
        result = []
        self._inorder(self.root, result)
        return result

    def _inorder(self, node, result):
        if node is None:
            return
        if node.is_leaf():
            result.append(node.key1)
            if node.is_3node():
                result.append(node.key2)
            return
        self._inorder(node.left, result)
        result.append(node.key1)
        self._inorder(node.mid, result)
        if node.is_3node():
            result.append(node.key2)
            self._inorder(node.right, result)

    def print_tree(self):
        if self.root is None:
            print("  (дерево порожнє)")
            return
        queue = [self.root]
        lvl   = 0
        while queue:
            print(f"  Рівень {lvl}: " + "   ".join(str(n) for n in queue))
            nxt = []
            for n in queue:
                if not n.is_leaf():
                    nxt.append(n.left)
                    nxt.append(n.mid)
                    if n.is_3node():
                        nxt.append(n.right)
            queue = nxt
            lvl  += 1

    def height(self):
        h, node = 0, self.root
        while node and not node.is_leaf():
            h += 1; node = node.left
        return h

def separator(title=""):
    print("\n" + "=" * 55)
    if title:
        print(f"  {title}")
        print("=" * 55)

def main():
    separator("2-3 ДЕРЕВО — організація пошукового індексу")

    print("\n[1] ВСТАВКА ключів: 9, 5, 8, 3, 2, 4, 7\n")

    keys = [9, 5, 8, 3, 2, 4, 7]

    tree = TwoThreeTree()
    for k in keys:
        tree.insert(k)
        print(f"  >> Вставлено {k}:")
        tree.print_tree()
        print()

    print("-" * 55)
    print("Фінальне дерево після всіх вставок:")
    tree.print_tree()
    print(f"\n  Висота: {tree.height()}")
    print(f"  In-order: {tree.inorder()}")

    separator("ДУБЛІКАТИ")
    print()
    dup_keys = [5, 3, 5, 8, 3, 3]
    print(f"  Вставляємо дублікати: {dup_keys}\n")
    for k in dup_keys:
        tree.insert(k)
        print(f"  >> Вставлено {k}  |  лічильники: {dict(sorted(tree.counts.items()))}")

    print("\n  Структура дерева НЕ змінилась (дублікати — лише в лічильнику):")
    tree.print_tree()
    print(f"\n  Лічильники входжень: {dict(sorted(tree.counts.items()))}")

    print("\n  Пошук ключів з урахуванням дублікатів:")
    for k in [3, 5, 8, 6]:
        node, cnt = tree.search(k)
        if node:
            print(f"    Ключ {k}: ЗНАЙДЕНО у {node},  входжень у індексі: {cnt}")
        else:
            print(f"    Ключ {k}: НЕ ЗНАЙДЕНО")

    separator("ВИДАЛЕННЯ")

    print("\n  [3.1] Видаляємо одне входження ключа 3 (було 4 копії):")
    tree.delete(3)
    print(f"  Лічильники: {dict(sorted(tree.counts.items()))}")
    print("  Структура дерева (без змін):")
    tree.print_tree()

    print("\n  [3.2] Видаляємо ключ 5 повністю (3 входження):")
    for _ in range(3):
        tree.delete(5)
    print(f"  Лічильники: {dict(sorted(tree.counts.items()))}")
    print("  Структура після видалення 5:")
    tree.print_tree()
    print(f"  In-order: {tree.inorder()}")

    print("\n  [3.3] Видаляємо ключ 2 (лист-2-вузол):")
    tree.delete(2)
    print("  Структура після видалення 2:")
    tree.print_tree()
    print(f"  In-order: {tree.inorder()}")

    print("\n  [3.4] Видаляємо ключ 8 (внутрішній вузол):")
    tree.delete(8)
    print("  Структура після видалення 8:")
    tree.print_tree()
    print(f"  In-order: {tree.inorder()}")

    print("\n  [3.5] Спроба видалити ключ 99 (якого немає):")
    result = tree.delete(99)
    print(f"  Результат: {'видалено' if result else 'не знайдено'}")

    separator("ФІНАЛЬНИЙ СТАН ІНДЕКСУ")
    print()
    tree.print_tree()
    print(f"\n  Унікальних ключів: {len(tree.inorder())}")
    print(f"  Лічильники: {dict(sorted(tree.counts.items()))}")
    print(f"  In-order:   {tree.inorder()}")
    print(f"  Висота:     {tree.height()}")
    print()


if __name__ == "__main__":
    main()
```

---

## Пояснення алгоритму

### Вставка

Новий ключ завжди вставляється у листовий вузол, який знаходиться шляхом спуску від кореня. Можливі два випадки:

- **Лист є 2-вузлом** - ключ додається, вузол стає 3-вузлом. Вставка завершена.
- **Лист є 3-вузлом** - вузол **розщеплюється**: з трьох ключів найменший залишається у поточному вузлі, найбільший переходить до нового вузла, а середній **піднімається** до батьківського вузла. Якщо батько теж є 3-вузлом - процес розщеплення повторюється вгору аж до кореня. Якщо корінь розщеплюється - дерево стає вищим на один рівень і створюється новий корінь.

### Пошук

Спуск від кореня до листа з порівняннями на кожному рівні:
- у 2-вузлі - одне порівняння, далі ліве або праве піддерево
- у 3-вузлі - до двох порівнянь, далі одне з трьох піддерев

### Видалення

Алгоритм складається з двох фаз:

1. **Зведення до листа**: якщо ключ знаходиться у внутрішньому вузлі, він замінюється in-order наступником (мінімальним ключем правого піддерева), після чого видаляється цей наступник з листа.

2. **Виправлення underflow**: після видалення ключа з листа-2-вузла утворюється порожній вузол. Батьківський вузол виправляє ситуацію одним із двох способів:
   - **Перерозподіл** - якщо братський вузол є 3-вузлом: роздільний ключ батька опускається до порожнього вузла, а один ключ брата піднімається до батька
   - **Злиття** - якщо братський вузол є 2-вузлом: порожній вузол, роздільний ключ батька і братський вузол зливаються в один 3-вузол. Якщо батько був 2-вузлом, він теж стає порожнім і underflow поширюється вгору

### Дублікати

Кожен ключ зберігається у дереві **один раз**. Кількість входжень відстежується у словнику `counts`. Це відповідає логіці реального пошукового індексу, де ключ - це слово, а лічильник - частота його появи у документах. Видалення зменшує лічильник; фізично ключ прибирається з дерева лише коли лічильник досягає нуля.

---

## Приклад роботи

Розглянемо вставку ключів: **9, 5, 8, 3, 2, 4, 7**

**Крок 1 - вставка 9:**
```
[9]
```

**Крок 2 - вставка 5:**
```
[5,9]       ← 2-вузол стає 3-вузлом
```

**Крок 3 - вставка 8:**
```
3-вузол [5,9] розщеплюється. 8 - середній → іде вгору як новий корінь

    [8]
   /   \
 [5]   [9]
```

**Крок 4 - вставка 3:**
```
    [8]
   /   \
 [3,5]  [9]
```

**Крок 5 - вставка 2:**
```
[2,3,5] - переповнення листа, розщеплення: 3 іде вгору

   [3,8]
  /  |  \
[2] [5]  [9]
```

**Крок 6 - вставка 4:**
```
   [3,8]
  /  |  \
[2] [4,5] [9]
```

**Крок 7 - вставка 7:**
```
[4,5,7] - переповнення → 5 іде вгору в [3,8]
[3,5,8] - переповнення кореня → 5 іде вгору як новий корінь

       [5]
      /   \
    [3]   [8]
   /  \  /  \
 [2] [4][7]  [9]
```

**Фінальне дерево (відповідає Рис. 2 у лекційному матеріалі)**

In-order обхід: **[2, 3, 4, 5, 7, 8, 9]** - відсортована послідовність ✓

**Приклад видалення ключа 5** (внутрішній вузол):

Замінюємо 5 на in-order наступника (7), потім видаляємо 7 з листа. Лист [7] - 2-вузол, виникає underflow. Братський вузол [9] - теж 2-вузол, виконується злиття. Результат:

```
      [3,8]
     /  |  \
   [2] [7]  [9]    (спрощений приклад для ілюстрації)
```

---

## Аналіз складності алгоритму

**Розмір вхідних даних**

Основний параметр - **N**, кількість ключів у дереві.

**Висота дерева**

Оскільки всі листя знаходяться на одному рівні, висота h задовольняє:

```
log₃(N+1) − 1  ≤  h  ≤  log₂(N+1) − 1
```

Звідси висота завжди дорівнює **Θ(log N)**.

**Часова складність**

| Операція | Найкращий | Середній | Найгірший |
|---|---|---|---|
| Пошук | O(log N) | O(log N) | O(log N) |
| Вставка | O(log N) | O(log N) | O(log N) |
| Видалення | O(log N) | O(log N) | O(log N) |

Всі три операції - **Θ(log N)** у будь-якому випадку, оскільки балансованість гарантована структурою. Це головна перевага над BST, у якого найгірший випадок - O(N).

**Просторова складність**

- Зберігання дерева: кожен з N ключів - в одному вузлі → **O(N)**
- Словник `counts` для дублікатів: **O(N)**
- Стек рекурсії (глибина = висота): **O(log N)**

Загальна просторова складність: **O(N)**

**Максимальний розмір задачі**

Інтерпретатор Python виконує приблизно 10⁷ операцій за секунду.

Оцінка для вставки N елементів (N вставок по O(log N) кожна):

```
N × log₂(N) ≤ 10⁷
```

При N = 700 000:  700 000 × log₂(700 000) ≈ 700 000 × 19,4 ≈ 13 600 000

При N = 500 000:  500 000 × 19,0 ≈ 9 500 000 ≈ 10⁷

Отже, за 1 секунду алгоритм може опрацювати близько **500 000 вставок**. Для пошуку та видалення - аналогічно.

**Зауваження**: одне ціле число в Python займає ~28 байт, вузол дерева (з посиланнями) - ~100–120 байт. При N = 500 000 дерево займе приблизно **55–60 МБ** пам'яті. Це прийнятний обсяг, тому для даного алгоритму визначальним є **процесор**, а не пам'ять.

---

## Додаткова інформація

**Порівняння з іншими структурами пошукового індексу**

| Структура | Пошук | Вставка | Видалення | Збалансованість |
|---|---|---|---|---|
| Масив (несортований) | O(N) | O(1) | O(N) | — |
| Відсортований масив | O(log N) | O(N) | O(N) | — |
| BST (звичайне) | O(log N) сер. / O(N) гірш. | O(log N) сер. | O(log N) сер. | ❌ |
| **2-3 дерево** | **O(log N)** | **O(log N)** | **O(log N)** | **✅** |
| Хеш-таблиця | O(1) сер. | O(1) сер. | O(1) сер. | — (немає порядку) |

2-3 дерево обирається тоді, коли критично важливі **гарантовані** часові характеристики і **впорядкованість** ключів (наприклад, для пошуку у діапазоні).

**Зв'язок з B-деревами**

2-3 дерево є частковим випадком **B-дерева** порядку 3. B-дерева широко застосовуються в системах баз даних і файлових системах (наприклад, NTFS, ext4), де вузол відповідає блоку диска і може містити сотні ключів.

---

## Посилання на онлайн компілятор:
https://www.jdoodle.com/ga/u%2B2xCJBA8lEJhsPWPkxKTQ%3D%3D
