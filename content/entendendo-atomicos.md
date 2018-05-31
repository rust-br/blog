+++
title = "Entendendo Atômicos"
description = "Explica o básico de atômicos em Rust. Vem com um bônus: implementação de spinlock."
date = 2018-05-30
tags = ["atômicos"]
category = "concorrência"
[extra]
author = "Bruno Corrêa Zimmermann"
+++

# Resumão

Neste post, explicarei o que são atômicos e como usá-los em Rust.

[English post in this link](./en/understanding-atomics.md)

# "De qualquer forma, o que seria um 'atômico'?"

Atômicos são tipos de dados com operações indivisíveis, ou as próprias
operações indivisíveis. Aqui indivisível significa que os efeitos dessas
operações só são observáveis quando a operação já acabou. Normalmente as
máquinas fornecem leitura e escrita tanto divisíveis quanto indivisíveis
sobre os inteiros nativos.

Se você já ouviu falar de "_data races_" (corrida de dados em tradução
literal), saiba que leitura e escrita divisíveis causam-nas. Suponha uma
máquina com palavra de 64 bits. Suponha _threads_ `A`, `B` e `C`, todas com
acesso ao endereço `p`. Se `A` faz uma escrita "divisível" de `x` em `p` ao
mesmo tempo em que `C` faz uma leitura "divisível", `C` pode ficar com metade
dos bits anteriores de `p` e metade dos bits de `x`. Ou um quarto desses bits.
Ou o estado anterior. Ou o novo estado. Na verdade, é difícil de prever. Se
a thread `A` escreve "divisivelmente" `y` de novo em `p` ao mesmo tempo em que
a thread `B` escreve "divisivelmente" `z`, o resultado de `p` pode ser uma
mistura de `y` e `z` ou algo similar.

Pelo fato do comportamento variar em diferentes máquinas, as linguagens de
programação como Rust e C++ definem _data races_ como comportamento indefinido
(_undefined behavior_). O comportamento real pode não ser "deixar um estado
intermediário", mas qualquer coisa, desde lançar uma exceção a botar fogo
no processador. Você provavelmente deve ter notado que o problema aqui são
as operações "divisíveis". Se `A` tivesse escrito `x` em `p` atomicamente e
`C` tivesse lido de `p` também atomicamente, `C` veria os bits de `p` ou antes
ou depois da escirta de `A`, mas nunca no meio. O mesmo se aplica a `A` e `B`
escrever atomicamente; o estado final seria ou `y` ou `z`, mas nada além disso.

Há ainda a chamada "_race condition_" (condição de corrida). Condição de
corrida é similar a _data race_, mas pode ocorrer até mesmo com atômicos.
Race condition é sobre dependência da ordem de eventos assíncronos, mas
não necessariamente corrompe dados. Condição de corrida é o que costuma
tornar estruturas de dados atômicas difíceis de serem escritas. Você
deve pensar com cuidado ao escrever código atômico.

# Atômicos em Rust

Atualmente (rust nightly 1.28 e stable 1.26) estes tipos de dados atômics
foram estabilizados: `AtomicPtr<T>`, `AtomicUsize`, `AtomicIsize` and
`AtomicBool`. Eles se localizam em `core::sync::atomic` (ou
`std::sync::atomic`). Pra explicar, escolhi o mais simples: `AtomicBool`.

## `AtomicBool`

É bastante simples de criar um `AtomicBool`:

```rust
let atomic_bool = AtomicBool::new(true);
```

Agora, vamos fazer leitura atômica! Mas espere... opa... `AtomicBool::load`
(carregar) aceita dois argumentos: uma referência imutável para `self` e um
argumento do tipo `Ordering`. A referência para `self` acredito de ser fácil de
se entender, mas e o `Ordering`? `Ordering` é um tipo que determina a... a...
ordem? Sim, a ordem relacionada às outras operações e às outras _threads_.
Veja abaixo a definição de `Ordering`:

```rust
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
    // some variants omitted
}
```

Há, de modo ríspido, dois tipos de CPUs relacionados às orderns: os "fracos"
e os "fortes". Os processadores "fracos" tem garantias naturalmente fracas a
respeito das ordens, enquanto os processadores "fortes" tem garantias
naturalmente fortes. Geralmente é de baixo custo para processadores "fortes"
executarem `Acquire`, `Release` e `AcqRel`, enquanto isso é de alto custo para
processadores "fracos". Para todos eles `Relaxed` é de baixo custo e `SeqCst`
é de alto custo. Exemplos de "fracos" são os das arquiteturas ARM em geral e
exemplos de fortes são os de arquiteturas x86/x86-64.

As únicas variantes válidas para se passar a `load` são: `Relaxed`. `Acquire` e
`SeqCst`. Não se preocupe, as outras variantes serão explicadas já já. `Relaxed`
é algo como "a ordem não importa mesmo". Isso permite tanto o CPU quanto o
compilador reordenarem a operação. `SeqCst` significa "sequencialmente
consistente"; e não será reordenada de jeito nenhum. Tudo antes da operação
acontence antes, e tudo depois da operação acontece depois.

`Acquire` é um pouco mais complexo. É o complementar de `Release`. Tudo
(geralmente operações no estilo `store` (armazenar)) que acontecem depois de
`Acquire` ficam depois do `Acquire`. Mas o compilador e o CPU são livres para
reordenar tudo que ocorre antes do `Acquire`. Ele foi projetado para ser usado
em operações do estilo `load` (carregar) enquanto adquirindo travas.

Vejamos um exemplo com `AtomicBool::load`:

```rust
use std::sync::atomic::{
    AtomicBool,
    Ordering::*,
};

fn print_if_it_is(atomic: &AtomicBool) {
    if atomic.load(Acquire) {
        println!("It is!");
    }
}
```

Agora vamos para a próxima operação: `store` (armazenar). `store` aceita uma
referência para `self`, o novo valor (do tipo `bool`) e um `Ordering`. Os
`Orderings`s válidos são `Relaxed`, `Release` e `SeqCst`. `Release`, como já
mencionado, é usado junto com `Acquire` como um par. Tudo antes de um `Release`
acontece antes dele, mas o CPU e o compilador são livres para reordenar tudo
que acontece depois. A intenção é usá-lo ao soltar uma trava.

Vejamos um exemplo:

```rust
use std::sync::atomic::{
    AtomicBool,
    Ordering::*,
};

fn store_something(atomic: &AtomicBool) {
    atomic.store(false, Release);
}
```

Você pode ter percebido duas coisas. Se não as percebeu, perceba agora.
Primeiro: `store` precisa de uma referencia apenas imutável. Isto significa que
ele não precisa de uma mutabilidade "externa", mas o atômico tem mutabilidade
interna. Segundo: atômicos só são úteis em referências compartilhadas. Eles não
tem nada como `clone`; para clonar você pode fazer algo como
`let copied = AtomicBool::new(src.load(Acquire));`. Mas isso não compartilha o
atômico. Desse modo, é comum encapsular o atômico (ou a estrutura que mantém
um) em um `Arc`.

Há pelo menos outras três operações importantes: `swap`, `compare_and_swap`,
`compare_exchange_*`. `swap` é bem simples: faz uma troca o argumento
(do tipo `bool`) com o valor armazenado, e retorna o valor armazenado. `swap`
aceita qualquer `Ordering`. `compare_and_swap` aceita um `bool`, dito "atual",
outro `bool`, dito "novo", e o `Ordering` (qualquer um é válido).
`compare_and_swap` é uma operação atômica bem importante. Ela compara o
argumento "atual" com o valor armazenado. Caso eles sejam iguais, a operação
troca o valor armazenado com o argumento "novo". De qualquer forma, o valor
armazenado é retornado. A operação foi um sucesso se o valor retornado é
igual ao argumento "atual".

Há ainda `compare_exchange_strong` (apenas `compare_exchange` em Rust). e
`compare_exchange_weak` (que pode falhar espontaneamente). Estes são similares
a `compare_and_swap`, exceto pelo fato de que aceitam duas ordens: uma para
o caso de sucesso e uma para o caso de falha. Além disso, em Rust elas retornam
um `Result`. O fato delas retornarem um `Result` possibilita
`compare_exchange_weak` falhar mesmo que a comparação seja um sucesso.
Importante: a ordem de sucesso aceita as mesmas variantes que `swap`, mas a de
falha tem de ser igual ou mais fraca do que a de sucesso, além de ser válida
para um `load`.

Calma aí! Eu não expliquei ainda `Ordering::AcqRel`. Ele é o que parece:
combina `Acquire` ao carregar e `Release` ao armazenar para operações que
carregam e armazenam atomicamente. Apesar de eu ter explicado tudo isso
usando `AtomicBool`, estas operações são bastante comuns entre tipos de
dados atômicos primitivos. Todos os tipos de dados atômicos do Rust têm
pelo menos estas operações: `load`, `store`, `swap`, `compare_and_swap`,
`compare_exchange` e `compare_exchange_weak`. Chega de conversa, vamos agir.

## "E" lógico livre de travas (lock-free)

A função abaixo recebe um booleano atômico, um booleano normal, armazena
o resultado de um "E" lógico entre eles e retorna o valor anterior.

```rust
use std::sync::atomic::{AtomicBool, Ordering::{*, self}};

fn lockfree_and(x: &AtomicBool, y: bool, ord: Ordering) -> bool {
    let mut stored = x.load(match ord {
        AcqRel => Acquire,
        Release => Relaxed,
        o => o,
    });
    loop {
        let inner = x.compare_and_swap(stored, stored & y, ord);
        if inner == stored {
            break inner;
        }
        stored = inner;
    }
}
```

Note que os efeitos da operação só são visíveis após acabarmos. Isto é, de
algum modo, "atômico". Geralmente, este tipo de operação é chamada "lock-free"
(livre de travas) porque ela não usa nenhuma forma de trava. Por mais que
tenhamos um loop de tentativas, nós não dependemos de nenhuma _thread_
liberando o recurso. Não é uma trava.

E, bem... é... Rust já fornece um método `AtomicBool::fetch_and`
("buscar 'e'"), que provavelmente é traduzida para uma única instrução nativa.
Eu quis apenas mostrar que você pode implementá-la com os primitivos `load` e
`compare_and_swap` via _software_.
[Dê uma olhada aqui](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.fetch_and),
Temos outras operações em `AtomicBool`, tais como `nand` ("não 'e'"), `or`
("ou") e `xor` ("ou exclusivo").

Como você deve suspeitar, `AtomicUsize` e `AtomicIsize` tem seus próprios
métodos com aritmética básica (adição e subtração) e bits. Todos eles são
implementáveis via software, mas provavelmente são traduzidos em uma única
instrução nativa. Apesar de a API não fornecer multiplicação e divisão,
não é difícil de implementá-las.

## Nota sobre `AtomicPtr<T>`

`AtomicPtr<T>` pode ser confuso. Os métodos `load` e `store` leem e escrevem
o endereço ou o conteúdo do endereço? A resposta é: o endereço. Eles operam
sobre um `*mut T`. Carregar o conteúdo é outra operação, que não é atômica e
é bastante _unsafe_.

## `Send` e `Sync`

Você talves tenha ouvido falar das _auto trait_s `Send` e `Sync`. Elas são
bastante importantes em ambientes concorrentes. `T: Send` significa que `T` é
seguro de ser mandado para outra _thread_. `T: Sync` significa que `&T` é
seguro de ser compartilhado entre _threads_. Tipos de dados atômicos
implementam tanto `Send` quanto `Sync`, então é seguro mandar e compartilhar
estes tipos através de _threads_.

## Bônus: implementação de spinlock

```rust
pub mod spinlock {
    use std::{
        cell::UnsafeCell,
        fmt,
        ops::{Deref, DerefMut},
        sync::atomic::{
            AtomicBool,
            Ordering::*,
        },
    };

    #[derive(Debug)]
    pub struct Mutex<T> {
        locked: AtomicBool,
        inner: UnsafeCell<T>,
    }

    #[derive(Debug, Clone, Copy)]
    pub struct MutexErr;

    pub struct MutexGuard<'a, T>
    where
        T: 'a
    {
        mutex: &'a Mutex<T>,
    }

    impl<T> Mutex<T> {
        pub fn new(data: T) -> Self {
            Self {
                locked: AtomicBool::new(false),
                inner: UnsafeCell::new(data),
            }
        }

        pub fn try_lock<'a>(&'a self) -> Result<MutexGuard<'a, T>, MutexErr> {
            if self.locked.swap(true, Acquire) {
                Err(MutexErr)
            } else {
                Ok(MutexGuard {
                    mutex: self,
                })
            }
        }

        pub fn lock<'a>(&'a self) -> MutexGuard<'a, T> {
            loop {
                if let Ok(m) = self.try_lock() {
                    break m;
                }
            }
        }
    }

    unsafe impl<T> Send for Mutex<T>
    where
        T: Send,
    {
    }

    unsafe impl<T> Sync for Mutex<T>
    where
        T: Send,
    {
    }

    impl<T> Drop for Mutex<T> {
        fn drop(&mut self) {
            unsafe {
                self.inner.get().drop_in_place()
            }
        }
    }

    impl<'a, T> Deref for MutexGuard<'a, T> {
        type Target = T;

        fn deref(&self) -> &T {
            unsafe {
                &*self.mutex.inner.get()
            }
        }
    }

    impl<'a, T> DerefMut for MutexGuard<'a, T> {
        fn deref_mut(&mut self) -> &mut T {
            unsafe {
                &mut *self.mutex.inner.get()
            }
        }
    }

    impl<'a, T> fmt::Debug for MutexGuard<'a, T>
    where
        T: fmt::Debug,
    {
        fn fmt(&self, fmtr: &mut fmt::Formatter) -> fmt::Result {
            write!(fmtr, "{:?}", &**self)
        }
    }

    impl<'a, T> Drop for MutexGuard<'a, T> {
        fn drop(&mut self) {
            let _prev = self.mutex.locked.swap(false, Release);
            debug_assert!(_prev);
        }
    }

}
```
