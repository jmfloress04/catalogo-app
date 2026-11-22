# Etapa builder
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
# --user deja todo en /root/.local y --no-cache-dir evita guardar basura de pip
RUN pip install --user --no-cache-dir -r requirements.txt

# Etapa final
FROM python:3.12-slim
WORKDIR /app
# crear usuario sin privilegios explicitamente ya que python no trae uno por defecto
RUN useradd --create-home --shell /bin/bash appuser
# copiar dependencias instaladas desde el builder al home de appuser
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
# copiar todo el codigo del backend
COPY --chown=appuser:appuser . .
# version horneada en la imagen - ARG solo en build - ENV persiste en el contenedor
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}
# ajusta el path para localizar uvicorn y desactivar el bufer de salida de python
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1
# cambia al usuario no root despues de los copy
USER appuser
# puerto sugerido
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
