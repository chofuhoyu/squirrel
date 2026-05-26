## Constraints

以下命令禁止直接运行。当你需要使用这些命令时，除非用户明确表示需要你做这些操作，否则需要先询问用户再决定。

- `git commit`
- `git push`

路径约定：所有路径使用 `~` 表示家目录（如 `~/Code/squirrel`、`~/Library/Rime`），不使用具体用户名。

搜索代码时，**禁止在家目录下全局搜索**。所有 `find`、`grep`、agent 搜索必须限定在项目目录内（`~/Code/squirrel` 或 `~/Library/Rime`），不能超出这些范围。

# Tips

始终通过`make xxx -j $(nproc)`的形式调用make以提高编译速度