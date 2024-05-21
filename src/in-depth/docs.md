# 為你的 CLI 程式產生文件 

CLI 的文件 通常包括指令中的 `--help` 部分和一個手冊（`man`）頁面。

Both can be automatically generated
when using [`clap`](https://crates.io/crates/clap), via
[`clap_mangen`](https://crates.io/crates/clap_mangen) crate.

兩者都可以自動生成
使用 [`clap`](https://crates.io/crates/clap) 時，透過[`clap_mangen`](https://crates.io/crates/clap_mangen) 箱。

```rust,ignore
#[derive(Parser)]
pub struct Head {
    /// file to load
    pub file: PathBuf,
    /// how many lines to print
    #[arg(short = "n", default_value = "5")]
    pub count: usize,
}
```

Secondly, you need to use a `build.rs`
to generate the manual file at compile time
from the definition of your app
in code.

There are a few things to keep in mind
(such as how you want to package your binary)
but for now
we simply put the `man` file
next to our `src` folder.

```rust,ignore
use clap::CommandFactory;

#[path="src/cli.rs"]
mod cli;

fn main() -> std::io::Result<()> {
    let out_dir = std::path::PathBuf::from(std::env::var_os("OUT_DIR").ok_or_else(|| std::io::ErrorKind::NotFound)?);
    let cmd = cli::Head::command();

    let man = clap_mangen::Man::new(cmd);
    let mut buffer: Vec<u8> = Default::default();
    man.render(&mut buffer)?;

    std::fs::write(out_dir.join("head.1"), buffer)?;

    Ok(())
}
```

When you now compile your application
there will be a `head.1` file
in your project directory.

If you open that in `man`
you'll be able to admire your free documentation.
如果你在 `man` 中打開它，則你可閱讀
