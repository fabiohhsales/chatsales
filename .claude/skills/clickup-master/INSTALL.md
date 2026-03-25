# Instalação do ClickUp MASTER Agent

A skill já está salva neste repositório. Para instalar localmente:

## Instalação (um único comando)

No CMD ou Terminal do Windows:

```cmd
mkdir "%USERPROFILE%\.claude\skills\clickup-master" && copy ".claude\skills\clickup-master\SKILL.md" "%USERPROFILE%\.claude\skills\clickup-master\SKILL.md"
```

Ou via Python (mais robusto):

```python
import os, shutil
dest = os.path.expanduser(r'~\.claude\skills\clickup-master')
os.makedirs(dest, exist_ok=True)
shutil.copy(r'.claude\skills\clickup-master\SKILL.md', os.path.join(dest, 'SKILL.md'))
print('Instalado em:', dest)
```

## Verificação

Após instalar, teste em uma nova conversa com:
- `"Cria tarefa no ClickUp: [título]"`
- `"Qual o status do projeto [X]?"`

A skill ativa automaticamente — sem restart necessário.

## Skills instaladas (openclaw/skills)

| Skill | Repositório |
|-------|-------------|
| clickup-operational | skills/aiwithabidi/clickup-operational/ |
| clickup-ticket-manager | skills/niyol/clickup-ticket-manager/ |
| clickup-project-management | skills/taazkareem/clickup-project-management/ |
