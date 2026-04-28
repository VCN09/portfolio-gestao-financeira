#  API de Gestão Financeira SaaS (Core Engine)

> **Status do Projeto:** Em Desenvolvimento Ativo (Sprint 4 - Módulo de Categorias & Multi-Tenancy)
> **Arquitetura:** Monolito Modular (Application Factory Pattern)
> **Foco:** Alta Performance, Segurança OWASP e Escalabilidade.
> **Badge:** ![Sprint 4](https://img.shields.io/badge/Sprint-4%20Completed-brightgreen)

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
    *   Hash de Senhas: Utilização de Bcrypt com salt adaptativo para garantir que senhas nunca sejam armazenadas em texto plano, mitigando riscos de vazamento de credenciais.
    *   Isolamento de Dados (Multi-Tenancy): Implementação de filtros obrigatórios por usuario_id em todas as queries, garantindo conformidade com a LGPD e prevenindo ataques de IDOR (Insecure Direct Object Reference).

##  Segurança e Arquitetura Multi-Tenant
O projeto implementa o conceito de **Privacy by Design**, garantindo o isolamento total de dados entre diferentes usuários (Tenants):

- **Autenticação Stateless:** Implementada via **JWT (JSON Web Tokens)** com expiração segura.
- **Criptografia de Senhas:** Utilização de **Bcrypt** para garantir que credenciais nunca sejam armazenadas em texto plano.
- **Isolamento de Dados (Multi-Tenancy):** Cada registro no banco de dados é vinculado a um `usuario_id` (UUID v4). Todas as consultas à API filtram automaticamente os resultados pelo ID do usuário autenticado, prevenindo vulnerabilidades de **IDOR**.

### Exemplo de Modelagem Protegida (SQLAlchemy):
```python
class Categoria(db.Model):
    id = db.Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    nome = db.Column(db.String(100), nullable=False)
    usuario_id = db.Column(UUID(as_uuid=True), db.ForeignKey('usuarios.id'), nullable=False)
    
    # O isolamento garante que Ana não acesse as categorias do Vitor
    usuario = db.relationship('Usuario', back_populates='categorias')

##  Arquitetura Multi-Tenant
Arquitetura Multi-Tenant: O sistema utiliza o modelo de banco de dados único com isolamento lógico. Cada registro é vinculado a um UUID de usuário, garantindo privacidade total entre as personas (Ex: "A", "B" e "C").

##  Modelagem de Dados (MER)

O núcleo do sistema foi modelado para suportar granularidade de despesas. A estrutura relacional (1:N) garante a integridade referencial:

*   `Usuario` (1) -> (N) `Despesa`
*   `Categoria` (1) -> (N) `Subcategoria`
*   `Subcategoria` (1) -> (N) `Despesa`

*(O diagrama ER completo está disponível na documentação interna do projeto no Miro).*

##  Snippet de Código

Implementação de Models, demonstrando o uso de UUIDs, Enums e validações em tempo de inicialização (Defense-in-Depth):
```python
import uuid
from sqlalchemy import Column, String, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID as PG_UUID
from sqlalchemy.orm import relationship, validates
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Usuario(Base):
    __tablename__ = 'usuarios'
    id = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    # outros campos como email, nome, etc.
    categorias = relationship("Categoria", back_populates="usuario")

class Categoria(Base):
    __tablename__ = 'categorias'

    id = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    nome = Column(String(100), nullable=False)
    usuario_id = Column(PG_UUID(as_uuid=True), ForeignKey('usuarios.id'), nullable=False)

    usuario = relationship("Usuario", back_populates="categorias")

    __table_args__ = (
        Index('ix_categorias_usuario_id', 'usuario_id'),
    )

    @validates('nome')
    def validate_nome(self, key, nome):
        if not nome or len(nome.strip()) == 0:
            raise ValueError('Nome da categoria não pode ser vazio.')
        if len(nome) > 100:
            raise ValueError('Nome da categoria deve ter no máximo 100 caracteres.')
        return nome.strip().title()

    # Demonstração de Multi-Tenant:
    # Em um ambiente multi-tenant, as consultas são sempre filtradas pelo usuario_id do tenant atual.
    # Exemplo:
    # categorias = session.query(Categoria).filter_by(usuario_id=current_user.id).all()
