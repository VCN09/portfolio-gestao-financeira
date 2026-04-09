#  API de Gestão Financeira SaaS (Core Engine)

> **Status do Projeto:** Em Desenvolvimento Ativo (Sprint 2)
> **Arquitetura:** Monolito Modular (Application Factory Pattern)
> **Foco:** Alta Performance, Segurança OWASP e Escalabilidade.

##  Visão Geral
Este repositório apresenta a fundação arquitetural de uma API RESTful desenvolvida em **Python (Flask)** para um sistema SaaS de Gestão Financeira Pessoal e Familiar. O projeto foi desenhado desde o dia zero visando escalabilidade comercial, suportando múltiplos perfis de usuários (CLT, PJ, MEI) e fluxos de caixa complexos.

*Nota: Por questões de segurança e propriedade intelectual, este repositório atua como um portfólio arquitetural. O código-fonte completo e as regras de negócio sensíveis residem em um repositório privado.*

##  Stack Tecnológico & Decisões Arquiteturais (ADRs)

A escolha de cada tecnologia foi pautada em resiliência e performance:

*   **Backend:** Python 3.10+ com **Flask** (utilizando o padrão *Application Factory* e *Blueprints* para modularização).
*   **Database:** **PostgreSQL**. Escolhido pela forte conformidade **ACID**, essencial para transações financeiras onde a perda de dados é inaceitável.
*   **ORM:** **SQLAlchemy**. Implementado com estratégias de *Eager Loading* (`lazy='joined'`) para mitigar o problema de N+1 queries.
*   **Segurança (OWASP):**
    *   **Autenticação:** **JWT** (JSON Web Tokens) via `Flask-JWT-Extended` para controle de sessão *stateless*.
    *   **Prevenção de IDOR:** Utilização estrita de **UUIDs** (v4) como Primary Keys em todas as tabelas, impedindo ataques de enumeração.
    *   **Sanitização de Payloads:** **Marshmallow** para validação rigorosa de dados de entrada/saída (Schemas).

##  Modelagem de Dados (MER)

O núcleo do sistema foi modelado para suportar granularidade de despesas. A estrutura relacional (1:N) garante a integridade referencial:

*   `Usuario` (1) -> (N) `Despesa`
*   `Categoria` (1) -> (N) `Subcategoria`
*   `Subcategoria` (1) -> (N) `Despesa`

*(O diagrama ER completo está disponível na documentação interna do projeto no Miro).*

##  Snippet de Código

Implementação de Models, demonstrando o uso de UUIDs, Enums e validações em tempo de inicialização (Defense-in-Depth):
```python
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import validates
import uuid
from app import db

class Categoria(db.Model):
    __tablename__ = 'categorias'
    
    id = db.Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    nome = db.Column(db.String(50), nullable=False, unique=True, index=True)
    tipo_despesa = db.Column(
        db.Enum('Fixa', 'Variável', name='tipo_despesa_enum'), 
        nullable=False, default='Variável'
    )
    ativo = db.Column(db.Boolean, default=True, nullable=False, index=True)

    # Relacionamento com Eager Loading
    subcategorias = db.relationship(
        'Subcategoria', 
        back_populates='categoria', 
        cascade='all, delete-orphan', 
        lazy='joined'
    )

    @validates('nome')
    def validate_nome(self, key, nome):
        if not nome or len(nome.strip()) == 0:
            raise ValueError("Nome da categoria não pode estar vazio")
        return nome.strip()
