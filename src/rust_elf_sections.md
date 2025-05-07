
```rust,compile_fail,noplayground
#[derive(Debug)]
pub struct Data {
    name: [char; 5],
    id: i32,
}

#[unsafe(link_section = "META_DATA_SECTION")]
pub static _MY_DATA_1: Data = Data {
    name: ['H', 'e', 'l', 'l', 'o'],
    id: 0x1,
};

unsafe extern "C" {
    static __start_META_DATA_SECTION: Data;
    static __stop_META_DATA_SECTION: Data;
}

fn main() {
    let data_section = unsafe {
        std::slice::from_raw_parts(
            &__start_META_DATA_SECTION,
            (&__stop_META_DATA_SECTION as *const Data)
                .offset_from_unsigned(&__start_META_DATA_SECTION as *const Data)
        )
    };
    for iter in data_section {
        println!("name:{:#?},id:{}", iter.name, iter.id);
    }
}

```

https://blog.rust-lang.org/2024/05/17/enabling-rust-lld-on-linux/#summary