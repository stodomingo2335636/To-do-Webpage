```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8" />
	<meta name="viewport" content="width=device-width, initial-scale=1.0" />
	<title>To-Do List</title>
	<style>
		* { box-sizing: border-box; }
		body {
			margin: 0;
			min-height: 100vh;
			padding: 48px 20px;
			font: 16px/1.5 system-ui, sans-serif;
			color: #20243a;
			background: linear-gradient(135deg, #eef2ff, #f8fafc);
		}
		.app {
			width: min(100%, 560px);
			margin: 0 auto;
			padding: 32px;
			background: #fff;
			border-radius: 20px;
			box-shadow: 0 18px 50px #20243a18;
		}
		h1 { margin: 0 0 6px; font-size: 2rem; }
		.subtitle { margin: 0 0 24px; color: #697087; }
		form { display: flex; gap: 10px; }
		input[type="text"] {
			flex: 1;
			min-width: 0;
			padding: 12px 14px;
			border: 1px solid #d9deea;
			border-radius: 10px;
			font: inherit;
		}
		button {
			padding: 10px 15px;
			border: 0;
			border-radius: 10px;
			color: #fff;
			background: #5755d9;
			font: inherit;
			font-weight: 600;
			cursor: pointer;
		}
		button:hover { background: #4543bf; }
		button:focus-visible, input:focus-visible { outline: 3px solid #aaa9ff; outline-offset: 2px; }
		.toolbar {
			display: flex;
			align-items: center;
			justify-content: space-between;
			gap: 12px;
			margin: 22px 0 10px;
		}
		.filters { display: flex; gap: 6px; }
		.filters button, .clear-button {
			padding: 7px 10px;
			color: #555d73;
			background: transparent;
			font-size: .9rem;
		}
		.filters button[aria-pressed="true"] { color: #4543bf; background: #eeeeff; }
		.clear-button { color: #b42335; }
		ul { list-style: none; padding: 0; margin: 0; }
		li {
			display: flex;
			align-items: center;
			gap: 12px;
			padding: 14px 4px;
			border-bottom: 1px solid #edf0f5;
		}
		li input { width: 18px; height: 18px; accent-color: #5755d9; }
		.task-text { flex: 1; overflow-wrap: anywhere; }
		li.completed .task-text { color: #9298a8; text-decoration: line-through; }
		.delete-button {
			padding: 6px 9px;
			color: #b42335;
			background: #fff1f2;
		}
		.empty { padding: 30px 8px; color: #7b8294; text-align: center; }
		.count { color: #697087; font-size: .9rem; }
		@media (max-width: 480px) {
			.app { padding: 24px 18px; }
			.toolbar { align-items: flex-start; flex-direction: column; }
		}
	</style>
</head>
<body>
	<main class="app">
		<h1>My To-Do List</h1>
		<p class="subtitle">A little progress adds up.</p>

		<form id="task-form">
			<label for="task-input" hidden>New task</label>
			<input id="task-input" type="text" placeholder="What needs to be done?" maxlength="200" required />
			<button type="submit">Add task</button>
		</form>

		<div class="toolbar">
			<span class="count" id="task-count" aria-live="polite"></span>
			<nav class="filters" aria-label="Filter tasks">
				<button type="button" data-filter="all" aria-pressed="true">All</button>
				<button type="button" data-filter="active" aria-pressed="false">Active</button>
				<button type="button" data-filter="completed" aria-pressed="false">Completed</button>
			</nav>
			<button class="clear-button" id="clear-completed" type="button">Clear completed</button>
		</div>

		<ul id="task-list"></ul>
		<p class="empty" id="empty-message" hidden>No tasks here. Add one above!</p>
	</main>

	<script>
		const STORAGE_KEY = "todo-list-tasks";
		const form = document.querySelector("#task-form");
		const input = document.querySelector("#task-input");
		const list = document.querySelector("#task-list");
		const count = document.querySelector("#task-count");
		const emptyMessage = document.querySelector("#empty-message");
		let filter = "all";
		let tasks = loadTasks();

		function loadTasks() {
			try {
				const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || "[]");
				return Array.isArray(saved) ? saved : [];
			} catch {
				return [];
			}
		}

		function saveTasks() {
			localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks));
		}

		function render() {
			list.replaceChildren();
			const visibleTasks = tasks.filter(task =>
				filter === "all" || (filter === "active" && !task.completed) ||
				(filter === "completed" && task.completed)
			);

			for (const task of visibleTasks) {
				const item = document.createElement("li");
				item.classList.toggle("completed", task.completed);

				const checkbox = document.createElement("input");
				checkbox.type = "checkbox";
				checkbox.checked = task.completed;
				checkbox.setAttribute("aria-label", `Mark ${task.text} as ${task.completed ? "active" : "completed"}`);
				checkbox.addEventListener("change", () => {
					task.completed = checkbox.checked;
					saveTasks();
					render();
				});

				const text = document.createElement("span");
				text.className = "task-text";
				text.textContent = task.text;

				const remove = document.createElement("button");
				remove.className = "delete-button";
				remove.type = "button";
				remove.textContent = "Delete";
				remove.setAttribute("aria-label", `Delete ${task.text}`);
				remove.addEventListener("click", () => {
					tasks = tasks.filter(item => item.id !== task.id);
					saveTasks();
					render();
				});

				item.append(checkbox, text, remove);
				list.append(item);
			}

			const activeCount = tasks.filter(task => !task.completed).length;
			count.textContent = `${activeCount} ${activeCount === 1 ? "task" : "tasks"} left`;
			emptyMessage.hidden = visibleTasks.length !== 0;
		}

		form.addEventListener("submit", event => {
			event.preventDefault();
			const text = input.value.trim();
			if (!text) return;
			tasks.push({ id: crypto.randomUUID(), text, completed: false });
			input.value = "";
			saveTasks();
			render();
			input.focus();
		});

		document.querySelectorAll("[data-filter]").forEach(button => {
			button.addEventListener("click", () => {
				filter = button.dataset.filter;
				document.querySelectorAll("[data-filter]").forEach(option =>
					option.setAttribute("aria-pressed", String(option === button))
				);
				render();
			});
		});

		document.querySelector("#clear-completed").addEventListener("click", () => {
			tasks = tasks.filter(task => !task.completed);
			saveTasks();
			render();
		});

		render();
	</script>
</body>
</html>
```
