import os
import sys
import io
import platform
import datetime
import tkinter as tk
from tkinter import filedialog, scrolledtext

class RustPyASTEngine:
    def __init__(self):
        self.lines = []
        self.idx = 0

    def tokenize(self, code):
        raw = code.splitlines()
        clean = []
        for l in raw:
            s = l.strip()
            if s and not s.startswith("//"):
                clean.append(s)
        return clean

    def parse_block(self):
        nodes = []
        while self.idx < len(self.lines):
            t = self.lines[self.idx]
            
            if t == "}":
                self.idx += 1
                return nodes

            if t.startswith("fn main"):
                self.idx += 1
                continue
                
            if t == "{":
                self.idx += 1
                continue

            if t.startswith("loops"):
                raise SyntaxError("Compile Error: Unknown keyword 'loops'. Did you mean 'loop'?")

            if t.startswith("let "):
                raw = t.replace(";", "")
                parts = raw[4:].split("=")
                var_name = parts[0].strip()
                if ":" in var_name:
                    var_name = var_name.split(":")[0].strip()
                val = parts[1].strip()
                
                if val == "true":
                    val = "True"
                elif val == "false":
                    val = "False"
                    
                nodes.append({"type": "assign", "var": var_name, "val": val})
                self.idx += 1
                continue

            if t.startswith("println!"):
                raw = t[:-1] if t.endswith(";") else t
                content = raw[9:-1].strip()
                nodes.append({"type": "print", "content": content})
                self.idx += 1
                continue

            if t.startswith("help(") or t.startswith("license(") or t.startswith("credits("):
                raw = t.replace(";", "").strip()
                cmd_type = "help_cmd"
                if raw.startswith("license"): cmd_type = "license_cmd"
                if raw.startswith("credits"): cmd_type = "credits_cmd"
                
                start_bracket = raw.find("(")
                param = raw[start_bracket+1:-1].strip().strip('"').strip("'")
                nodes.append({"type": cmd_type, "param": param if param else None})
                self.idx += 1
                continue

            if t == "loop":
                self.idx += 1
                body = self.parse_block()
                nodes.append({"type": "loop", "body": body})
                continue

            if t.startswith("if "):
                cond = t[3:].strip()
                if cond == "true": cond = "True"
                if cond == "false": cond = "False"
                self.idx += 1
                body = self.parse_block()
                nodes.append({"type": "if", "cond": cond, "body": body})
                continue

            if t.startswith("else if "):
                cond = t[8:].strip()
                if cond == "true": cond = "True"
                if cond == "false": cond = "False"
                self.idx += 1
                body = self.parse_block()
                nodes.append({"type": "elif", "cond": cond, "body": body})
                continue

            if t == "else":
                self.idx += 1
                body = self.parse_block()
                nodes.append({"type": "else", "body": body})
                continue

            if t == "break;" or t == "break":
                nodes.append({"type": "break"})
                self.idx += 1
                return nodes

            self.idx += 1
        return nodes

    def generate(self, nodes, level=0):
        py_code = []
        indent = "    " * level
        
        for idx, n in enumerate(nodes):
            if n["type"] == "assign":
                py_code.append(f"{indent}_rust_assign('{n['var']}', {n['val']})")
            
            elif n["type"] == "help_cmd":
                if n["param"]:
                    py_code.append(f"{indent}help('{n['param']}')")
                else:
                    py_code.append(f"{indent}help()")
                    
            elif n["type"] == "license_cmd":
                py_code.append(f"{indent}license()")
                
            elif n["type"] == "credits_cmd":
                py_code.append(f"{indent}credits()")

            elif n["type"] == "print":
                raw_content = n["content"]
                if "," in raw_content:
                    fmt, args = raw_content.split(",", 1)
                    fmt = fmt.strip().strip('"')
                    args = [a.strip() for a in args.split(",")]
                    for a in args:
                        fmt = fmt.replace("{}", f"{{_rust_expr('{a}')}}", 1)
                    py_code.append(f'{indent}print(f"{fmt}")')
                else:
                    py_code.append(f"{indent}print({raw_content})")
            
            elif n["type"] == "loop":
                py_code.append(f"{indent}while True:")
                py_code.append(self.generate(n["body"], level + 1))
            
            elif n["type"] == "if":
                py_code.append(f"{indent}if _rust_expr('{n['cond']}'):")
                py_code.append(self.generate(n["body"], level + 1))

            elif n["type"] == "elif":
                py_code.append(f"{indent}elif _rust_expr('{n['cond']}'):")
                py_code.append(self.generate(n["body"], level + 1))

            elif n["type"] == "else":
                py_code.append(f"{indent}else:")
                py_code.append(self.generate(n["body"], level + 1))
            
            elif n["type"] == "break":
                py_code.append(f"{indent}break")
                
        return "\n".join(py_code)

    def run_transpiler(self, code):
        self.lines = self.tokenize(code)
        self.idx = 0
        ast = self.parse_block()
        return self.generate(ast)


class RustPyIDLE:
    def __init__(self):
        self.compiler = RustPyASTEngine()
        self.root = tk.Tk()
        self.root.title("RustPy IDLE - v1.3.0")
        self.root.geometry("830x650")
        
        self.is_dark_mode = False

        self.control_bar = tk.Frame(self.root, height=30, bg="#f0f0f0")
        self.control_bar.pack(fill=tk.X, side=tk.TOP)
        
        self.theme_btn = tk.Button(self.control_bar, text="◐ Dark/Light", font=("Arial", 9), command=self.toggle_theme, bd=1, relief="raised")
        self.theme_btn.pack(side=tk.RIGHT, padx=5, pady=2)

        self.menu = tk.Menu(self.root)
        self.file_menu = tk.Menu(self.menu, tearoff=0)
        self.file_menu.add_command(label="Save As...", command=self.save)
        self.file_menu.add_separator()
        self.file_menu.add_command(label="Exit", command=self.root.destroy)
        self.menu.add_cascade(label="File", menu=self.file_menu)
        
        self.run_menu = tk.Menu(self.menu, tearoff=0)
        self.run_menu.add_command(label="Run Module", command=self.execute, accelerator="F5")
        self.menu.add_cascade(label="Run", menu=self.run_menu)
        self.root.config(menu=self.menu)
        self.root.bind("<F5>", lambda e: self.execute())

        self.editor = scrolledtext.ScrolledText(self.root, font=("Courier New", 11), bd=2, relief="sunken", undo=True)
        self.editor.pack(fill=tk.BOTH, expand=True, padx=4, pady=4)
        
        self.editor.tag_config("keyword", foreground="#0000FF", font=("Courier New", 11, "bold"))
        self.editor.tag_config("type", foreground="#2B818A", font=("Courier New", 11))
        self.editor.tag_config("macro", foreground="#A31515", font=("Courier New", 11))
        self.editor.tag_config("main_word", foreground="#795E26", font=("Courier New", 11, "bold"))
        
        self.editor.bind("<KeyRelease>", lambda e: self.apply_highlighting())

        snippet = 'fn main()\n{\n    help();\n    credits();\n    license();\n}'
        self.editor.insert(tk.END, snippet)

        self.terminal = scrolledtext.ScrolledText(self.root, font=("Courier New", 10), bg="#ffffff", fg="#000000", bd=2, relief="groove", height=14)
        self.terminal.pack(fill=tk.X, side=tk.BOTTOM, padx=4, pady=4)
        
        self.apply_highlighting()
        self.display_boot_diagnostics()

    def toggle_theme(self):
        if not self.is_dark_mode:
            self.editor.config(bg="#1E1E1E", fg="#D4D4D4", insertbackground="white")
            self.terminal.config(bg="#1E1E1E", fg="#00FF00", insertbackground="white")
            self.control_bar.config(bg="#2D2D2D")
            self.is_dark_mode = True
        else:
            self.editor.config(bg="#FFFFFF", fg="#000000", insertbackground="black")
            self.terminal.config(bg="#FFFFFF", fg="#000000", insertbackground="black")
            self.control_bar.config(bg="#f0f0f0")
            self.is_dark_mode = False
        self.apply_highlighting()

    def display_boot_diagnostics(self):
        now = datetime.datetime.now()
        day_str = now.strftime("%A")
        date_str = now.strftime("%Y-%m-%d")
        time_str = now.strftime("%H:%M:%S")
        
        arch_info = platform.architecture()[0]
        machine_type = platform.machine()
        
        self.terminal.config(state=tk.NORMAL)
        self.terminal.insert(tk.END, f"==================================================\n")
        self.terminal.insert(tk.END, f" RUSTPY(RUST-PYTHON IDLE) ENVIRONMENT INITIALIZED\n")
        self.terminal.insert(tk.END, f"--------------------------------------------------\n")
        self.terminal.insert(tk.END, f" TIMESTAMP    : {day_str}, {date_str} [{time_str}]\n")
        self.terminal.insert(tk.END, f" CORE ENGINE  : {machine_type} ({arch_info} architecture)\n")
        self.terminal.insert(tk.END, f" STACK OWNER  : Kev1inmates Core Transpiler Build\n")
        self.terminal.insert(tk.END, f"--------------------------------------------------\n")
        self.terminal.insert(tk.END, f" -> Type help(); credits(); or license(); inside\n")
        self.terminal.insert(tk.END, f"    your source script to fetch environment diagnostic info.\n")
        self.terminal.insert(tk.END, f"==================================================\n\n")
        self.terminal.config(state=tk.DISABLED)

    def apply_highlighting(self):
        for tag in ["keyword", "type", "macro", "main_word"]:
            self.editor.tag_remove(tag, "1.0", tk.END)
            
        content = self.editor.get("1.0", tk.END)
        lines = content.splitlines()
        
        keywords = ["if", "else if", "else", "loop", "let"]
        types = ["int", "str", "float", "bool", "chr"]
        macros = ["println!", "printf", "help", "credits", "license"]
        
        kw_color = "#569CD6" if self.is_dark_mode else "#0000FF"
        type_color = "#4EC9B0" if self.is_dark_mode else "#2B818A"
        macro_color = "#DCDCAA" if self.is_dark_mode else "#A31515"
        main_color = "#DCDCAA" if self.is_dark_mode else "#795E26"
        
        self.editor.tag_config("keyword", foreground=kw_color)
        self.editor.tag_config("type", foreground=type_color)
        self.editor.tag_config("macro", foreground=macro_color)
        self.editor.tag_config("main_word", foreground=main_color)
        
        for row, line in enumerate(lines, start=1):
            if "main" in line:
                start_idx = line.find("main")
                end_idx = start_idx + 4
                self.editor.tag_add("main_word", f"{row}.{start_idx}", f"{row}.{end_idx}")

            for kw in keywords:
                start_idx = 0
                while True:
                    start_idx = line.find(kw, start_idx)
                    if start_idx == -1:
                        break
                    end_idx = start_idx + len(kw)
                    self.editor.tag_add("keyword", f"{row}.{start_idx}", f"{row}.{end_idx}")
                    start_idx = end_idx
                    
            for t in types:
                start_idx = 0
                while True:
                    start_idx = line.find(t, start_idx)
                    if start_idx == -1:
                        break
                    end_idx = start_idx + len(t)
                    self.editor.tag_add("type", f"{row}.{start_idx}", f"{row}.{end_idx}")
                    start_idx = end_idx
                    
            for m in macros:
                start_idx = 0
                while True:
                    start_idx = line.find(m, start_idx)
                    if start_idx == -1:
                        break
                    end_idx = start_idx + len(m)
                    self.editor.tag_add("macro", f"{row}.{start_idx}", f"{row}.{end_idx}")
                    start_idx = end_idx

    def save(self):
        p = filedialog.asksavesasfilename(defaultextension=".rpy", filetypes=[("RustPy Source", "*.rpy")])
        if p:
            with open(p, "w") as f:
                f.write(self.editor.get("1.0", tk.END))

    def run_help_system(self, topic=None):
        if topic is None:
            print("=== RustPy Help Menu ===")
            print("Keywords you can look up:")
            print(" -> help(\"let\")")
            print(" -> help(\"println\")")
            print(" -> help(\"if\")")
            print(" -> help(\"else\")")
            print(" -> help(\"loop\")")
            print("========================")
            return

        topic = str(topic).strip().lower()
        if topic == "let":
            print("[HELP]: Creates a variable. Example: let x = 10;")
        elif topic == "println":
            print("[HELP]: Prints text out to the console.")
        elif topic == "if":
            print("[HELP]: Checks conditions using if, else if, and else blocks.")
        elif topic == "else":
            print("[HELP]: Executes a default block when all previous 'if' conditions fail.")
        elif topic == "loop":
            print("[HELP]: Runs an infinite loop block until a break is hit.")
        else:
            print(f"[HELP]: Unknown word: '{topic}'")

    def display_license_info(self):
        print("\n--- MIT LICENSE COPYRIGHT STACK ---")
        print("Copyright (c) 2026 Kev1inmates (Discord)")
        print("Permission is hereby granted, free of charge, to any person obtaining a copy")
        print("of this software and associated documentation files (the 'Software'), to deal")
        print("in the Software without restriction, including without limitation the rights")
        print("to use, copy, modify, merge, publish, distribute, sublicense, and/or sell")
        print("copies of the Software...")
        print("-----------------------------------")

    def display_credits_info(self):
        print("\n--- COMPILER PACKAGE CREDITS ---")
        print("Lead Core Architect : Kev1inmates (Discord)")
        print("Target Build Engine : RustPy Script System")
        print("Current Environment : Production-v1.3.0 Stable")
        print("---------------------------------")

    def execute(self):
        src = self.editor.get("1.0", tk.END)
        self.terminal.config(state=tk.NORMAL)

        try:
            py_out = self.compiler.run_transpiler(src)
            buf = io.StringIO()
            sys.stdout = buf
            
            _scope = {}
            def _rust_assign(name, val):
                _scope[name] = val
            def _rust_expr(expr_str):
                return eval(expr_str, {}, _scope)

            runtime_env = {
                "_rust_assign": _rust_assign,
                "_rust_expr": _rust_expr,
                "print": print,
                "help": self.run_help_system,
                "license": self.display_license_info,
                "credits": self.display_credits_info,
                "True": True,
                "False": False
            }
            
            exec(py_out, runtime_env)
            sys.stdout = sys.__stdout__
            self.terminal.insert(tk.END, buf.getvalue())
        except Exception as ex:
            sys.stdout = sys.__stdout__
            self.terminal.insert(tk.END, f"{ex}\n")
        
        self.terminal.config(state=tk.DISABLED)

if __name__ == "__main__":
    app = RustPyIDLE()
    app.root.mainloop()
