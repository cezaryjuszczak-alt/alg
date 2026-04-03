### Iterator listy jednokierunkowej

Lista jednokierunkowa to struktura danych, w której każdy węzeł zawiera dane oraz wskaźnik do następnego węzła, a ostatni węzeł nie wskazuje na nic. W Rustcie, z uwagi na zarządzanie pamięcią i bezpieczeństwo, użyjemy smart pointera **Box** oraz typu **Option** do reprezentacji tej struktury. Iterator będzie przechodził przez listę, zwracając referencje do danych w każdym węźle.

Najpierw zdefiniujmy struktury dla węzła i listy:

rust

```rust
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>,
}

struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
}
```

- **Node<T>**: Reprezentuje pojedynczy węzeł listy.
  - data: T przechowuje dane typu T.
  - next: Option<Box<Node<T>>> wskazuje na następny węzeł lub None, jeśli jest to ostatni węzeł. Box alokuje węzeł na stercie, a Option pozwala na brak następnika.
- **LinkedList<T>**: Reprezentuje całą listę.
  - head: Option<Box<Node<T>>> to początek listy, który może być pusty (None).

##### Struktura iteratora

Iterator potrzebuje śledzić bieżącą pozycję w liście. Ponieważ będzie on jedynie pożyczał dane z listy (nie przejmował jej na własność), użyjemy referencji z parametrem życia 'a:

rust

```rust
struct LinkedListIter<'a, T> {
    current: Option<&'a Node<T>>,
}
```

- **LinkedListIter<'a, T>**: Struktura iteratora.
  - current: Option<&'a Node<T>> przechowuje referencję do bieżącego węzła lub None, jeśli iterator dotarł do końca listy.
  - 'a to parametr życia, który wiąże iterator z życiem listy, którą iteruje.

#### Implementacja traitu Iterator

W Rustcie iterator definiuje się poprzez implementację traitu Iterator, który wymaga metody next. Metoda ta zwraca kolejny element (lub None, jeśli nie ma więcej elementów):

rust

```rust
impl<'a, T> Iterator for LinkedListIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        match self.current {
            Some(node) => {
                self.current = node.next.as_deref();
                Some(&node.data)
            }
            None => None,
        }
    }
}
```

- **type Item = &'a T**: Określa, że iterator zwraca referencje do danych (&'a T), co jest typowe dla iteratorów pożyczających.
- **fn next(&mut self) -> Option<Self::Item>**:
  - Jeśli self.current to Some(node), pobieramy dane (&node.data) i przesuwamy self.current na następny węzeł za pomocą node.next.as_deref(). as_deref() konwertuje &Option<Box<Node<T>>> na Option<&Node<T>>.
  - Jeśli self.current to None, zwracamy None, co oznacza koniec iteracji.

##### Metoda iter na liście

Aby utworzyć iterator z listy, dodajemy metodę iter do LinkedList:

rust

```rust
impl<T> LinkedList<T> {
    fn iter<'a>(&'a self) -> LinkedListIter<'a, T> {
        LinkedListIter {
            current: self.head.as_deref(),
        }
    }
}
```

- **fn iter<'a>(&'a self) -> LinkedListIter<'a, T>**:
  - Przyjmuje referencję do listy (&'a self) i zwraca iterator.
  - self.head.as_deref() konwertuje Option<Box<Node<T>>> na Option<&Node<T>>, co pasuje do typu current w LinkedListIter.
  - Parametr życia 'a zapewnia, że iterator jest ważny tak długo, jak referencja do listy.

##### Implementacja IntoIterator

Aby lista mogła być używana w pętlach for, implementujemy trait IntoIterator dla referencji do listy:

rust

```rust
impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type Item = &'a T;
    type IntoIter = LinkedListIter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}
```

- **type Item = &'a T**: Iterator zwraca referencje do danych.
- **type IntoIter = LinkedListIter<'a, T>**: Typ iteratora zwracanego przez into_iter.
- **fn into_iter(self) -> Self::IntoIter**:
  - self to &'a LinkedList<T>, więc wywołujemy metodę iter(), która zwraca LinkedListIter<'a, T>.

##### Pełny kod

Oto kompletna implementacja:

rust

```rust
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>,
}

struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
}

struct LinkedListIter<'a, T> {
    current: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for LinkedListIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        match self.current {
            Some(node) => {
                self.current = node.next.as_deref();
                Some(&node.data)
            }
            None => None,
        }
    }
}

impl<T> LinkedList<T> {
    fn iter<'a>(&'a self) -> LinkedListIter<'a, T> {
        LinkedListIter {
            current: self.head.as_deref(),
        }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type Item = &'a T;
    type IntoIter = LinkedListIter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}
```

Wyjaśnienie działania

- **Pusta lista**: Jeśli head jest None, iter() ustawi current na None, a next() od razu zwróci None.
- **Lista z jednym elementem**: head wskazuje na węzeł z next = None. Iterator zwróci dane pierwszego węzła, a następnie None.
- **Lista z wieloma elementami**: Iterator przejdzie przez wszystkie węzły, zwracając dane w kolejności, aż dojdzie do końca (current = None).

Ta implementacja zapewnia bezpieczne i idiomatyczne podejście do iteracji po liście jednokierunkowej w Rustcie, zgodne z zasadami pożyczania i zarządzania pamięcią. Iterator jest typu pożyczającego, co jest standardem w bibliotekach Rust, takich jak Vec czy HashMap, gdy iterujemy po referencjach.



#### Iterator mutowalny do listy

Aby dodać metodę iter_mut do listy jednokierunkowej w Rust, która umożliwi iterację z możliwością modyfikacji elementów, musimy zdefiniować nowy typ iteratora zwracającego mutowalne referencje (&mut T) oraz zaimplementować odpowiednie trait-y. Poniżej przedstawiam kompletną implementację wraz z wyjaśnieniami.

------

Krok 1: Definicja struktury listy i węzła

Zacznijmy od definicji podstawowych struktur: Node (węzeł) i LinkedList (lista).

rust

```rust
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>,
}

struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
}
```

- Node<T> zawiera dane (data) i opcjonalny wskaźnik na następny węzeł (next).
- LinkedList<T> przechowuje tylko głowę listy (head).

------

Krok 2: Definicja iteratora mutowalnego

Teraz definiujemy strukturę dla iteratora mutowalnego, który będzie przechowywał mutowalną referencję do bieżącego węzła:

rust

```rust
struct LinkedListIterMut<'a, T> {
    current: Option<&'a mut Node<T>>,
}
```

- LinkedListIterMut<'a, T> to iterator mutowalny.
- Pole current typu Option<&'a mut Node<T>> przechowuje mutowalną referencję do bieżącego węzła lub None, jeśli iterator dotarł do końca listy.
- 'a to parametr życia, który zapewnia, że iterator jest ważny tak długo, jak lista, którą iteruje.

------

Krok 3: Implementacja traitu Iterator dla LinkedListIterMut

Musimy zaimplementować trait Iterator, aby iterator mógł zwracać mutowalne referencje do danych:

rust

```rust
impl<'a, T> Iterator for LinkedListIterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.current.take().map(|node| {
            self.current = node.next.as_deref_mut();
            &mut node.data
        })
    }
}
```

- type Item = &'a mut T określa, że iterator zwraca mutowalne referencje do danych.
- Metoda next:
  - Używa take(), aby wyjąć wartość z self.current i ustawić je na None. Jest to wymagane, aby zapewnić unikalność mutowalnych referencji w Rust.
  - Jeśli self.current zawiera Some(node):
    - Ustawia self.current na mutowalną referencję do następnego węzła za pomocą node.next.as_deref_mut() (konwertuje &mut Option<Box<Node<T>>> na Option<&mut Node<T>>).
    - Zwraca mutowalną referencję do danych bieżącego węzła (&mut node.data).
  - Jeśli self.current to None, zwraca None.

------

Krok 4: Dodanie metody iter_mut do LinkedList

Dodajemy metodę iter_mut do struktury LinkedList, która zwraca instancję LinkedListIterMut:

rust

```rust
impl<T> LinkedList<T> {
    fn iter_mut<'a>(&'a mut self) -> LinkedListIterMut<'a, T> {
        LinkedListIterMut {
            current: self.head.as_deref_mut(),
        }
    }
}
```

- Metoda iter_mut przyjmuje mutowalną referencję do listy (&'a mut self), co pozwala na modyfikację jej elementów.
- self.head.as_deref_mut() konwertuje &mut Option<Box<Node<T>>> na Option<&mut Node<T>>, co jest zgodne z typem current w LinkedListIterMut.

------

Krok 5: Implementacja IntoIterator dla mutowalnych referencji

Aby umożliwić używanie iteratora w pętlach for, implementujemy trait IntoIterator dla &mut LinkedList<T>:

rust

```rust
impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type Item = &'a mut T;
    type IntoIter = LinkedListIterMut<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}
```

- type Item = &'a mut T określa, że iterator zwraca mutowalne referencje.
- type IntoIter = LinkedListIterMut<'a, T> definiuje typ zwracanego iteratora.
- into_iter wywołuje metodę iter_mut na mutowalnej referencji do listy.

------

Pełny kod

Oto kompletna implementacja z dodaną metodą iter_mut:

rust

```rust
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>,
}

struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
}

struct LinkedListIterMut<'a, T> {
    current: Option<&'a mut Node<T>>,
}

impl<'a, T> Iterator for LinkedListIterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.current.take().map(|node| {
            self.current = node.next.as_deref_mut();
            &mut node.data
        })
    }
}

impl<T> LinkedList<T> {
    fn iter_mut<'a>(&'a mut self) -> LinkedListIterMut<'a, T> {
        LinkedListIterMut {
            current: self.head.as_deref_mut(),
        }
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type Item = &'a mut T;
    type IntoIter = LinkedListIterMut<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}
```

------

#### Przykład użycia

Możemy teraz użyć metody iter_mut lub pętli for, aby modyfikować elementy listy:

rust

```rust
fn main() {
    let mut list = LinkedList {
        head: Some(Box::new(Node {
            data: 1,
            next: Some(Box::new(Node {
                data: 2,
                next: None,
            })),
        })),
    };

    // Użycie w pętli for
    for value in &mut list {
        *value += 1;
    }

    // Weryfikacja (tu mamy metodę iter() lub ręczne przejście)
    // Wartości w liście powinny być teraz 2 i 3
}
```

W tym przykładzie wartości w liście zostaną zwiększone o 1, co pokazuje, że iter_mut działa poprawnie, umożliwiając mutację elementów podczas iteracji.

------

#### Podsumowanie

Metoda iter_mut została dodana poprzez:

1. Zdefiniowanie struktury LinkedListIterMut dla iteratora mutowalnego.
2. Implementację traitu Iterator, który zwraca &mut T.
3. Dodanie metody iter_mut do LinkedList.
4. Implementację IntoIterator dla &mut LinkedList<T>, aby umożliwić użycie w pętlach for.

Ta implementacja jest bezpieczna dzięki systemowi własności i pożyczania w Rust, zapewniając, że mutowalne referencje są unikalne i prawidłowo zarządzane.
