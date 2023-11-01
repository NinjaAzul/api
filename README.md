# api

```ts

import { type FieldPath, type UseFormSetError } from "react-hook-form";

// import { redirect } from "next/navigation";

// import { toast } from "@autominer/ui";

import { clientCookies, serverCookies } from "@Application/utils";
import { ENVIRONMENTS } from "@Application/constants";

export interface IFetcherResponse<T> {
  data: T;
  success: boolean;
  status: number;
}

type FetcherOptions = Parameters<typeof fetch>[1] & {
  query?: Record<string, string | number | boolean | null | undefined>;
};

export class Fetcher {
  constructor(
    readonly baseUrl?: string,
    private readonly options?: { withoutInterceptors?: boolean },
  ) {}

  private getFormattedURL(url: Parameters<typeof fetch>[0], options?: FetcherOptions) {
    const baseURL = this.baseUrl || "";

    const urlFormatted = url.toString().startsWith("/") ? `${url}` : `/${url}`;

    if (options?.query) {
      const query = new URLSearchParams();
      Object.entries(options.query).forEach(([key, value]) => {
        if (value) query.append(key, value.toString());
      });
      return `${baseURL}${urlFormatted}?${query}`;
    }

    return `${baseURL}${urlFormatted}`;
  }

  private async requestInterceptor(url: Parameters<typeof fetch>[0], options: FetcherOptions = {}) {
    if (!options?.cache) Object.assign(options, { cache: "no-cache" });

    if (!this.options?.withoutInterceptors) {
      if (!options.headers) options.headers = {};

      const isServer = typeof window === "undefined";

      if (isServer) {
        const token = await serverCookies.getAuthCookie();

        Object.assign(options.headers, { Authorization: `Bearer ${token}` });
      } else {
        const token = clientCookies.getAuthCookie();
        Object.assign(options.headers, { Authorization: `Bearer ${token}` });
      }

      const hasContentTypeHeader =
        options.headers && Object.keys(options.headers).includes("Content-Type");

      const isFormData = options?.body instanceof FormData;

      if (!hasContentTypeHeader && !isFormData) {
        Object.assign(options.headers, { "Content-Type": "application/json" });
      }

      return fetch(this.getFormattedURL(url, options), options);
    } else {
      return fetch(this.getFormattedURL(url, options), options);
    }
  }

  private async request<T = unknown>(
    url: Parameters<typeof fetch>[0],
    options: FetcherOptions,
  ): Promise<IFetcherResponse<T>> {
    const response = await this.requestInterceptor(url, options);
    return this.responseInterceptor<T>(response);
  }

  private async responseInterceptor<T = unknown>(response: Response) {
    const data = (await response.json()) as T;

    // if (!this.options?.withoutInterceptors) {
    //   if (response.status === 401) {
    //     if (typeof window !== "undefined") {
    //       clientCookies.removeAuthCookie();
    //       alert("Você foi desconectado, por favor faça login novamente");
    //       redirect("/login");
    //     } else {
    //       await serverCookies.removeAuthCookie();
    //       redirect("/login");
    //     }
    //   }
    // }

    if (response.ok) return { data, success: response.ok, status: response.status };

    const error = data as any;
    throw new FetcherError(error.message, response.status, error);
  }

  async get<T = unknown>(url: Parameters<typeof fetch>[0], options: FetcherOptions = ({} = {})) {
    Object.assign(options, { ...options, method: "GET" });
    return this.request<T>(url, options);
  }

  async post<T = unknown>(url: Parameters<typeof fetch>[0], options: FetcherOptions = ({} = {})) {
    Object.assign(options, { ...options, method: "POST" });
    return this.request<T>(url, options);
  }

  async put<T = unknown>(url: Parameters<typeof fetch>[0], options: FetcherOptions = ({} = {})) {
    Object.assign(options, { ...options, method: "PUT" });
    return this.request<T>(url, options);
  }

  async patch<T = unknown>(url: Parameters<typeof fetch>[0], options: FetcherOptions = ({} = {})) {
    Object.assign(options, { ...options, method: "PATCH" });
    return this.request<T>(url, options);
  }

  async delete<T = unknown>(url: Parameters<typeof fetch>[0], options: FetcherOptions = ({} = {})) {
    Object.assign(options, { ...options, method: "DELETE" });
    return this.request<T>(url, options);
  }
}

export class FetcherError extends Error {
  constructor(
    public message: string,
    public status: number,
    public data?: FetchErrorResponse,
  ) {
    super(message);
    this.name = "FetcherError";
    this.status = status;
    this.data = data;
  }

  //@ts-ignore
  formatError<T>(setError: UseFormSetError<T>) {
    if (this.data?.errors) {
      this.data.errors.forEach((error) => {
        if (error.path) {
          //@ts-ignore
          setError(error.path as FieldPath<T>, {
            message: error.message,
            type: "manual",
          });
        }
      });
    }
  }
}

export const api = new Fetcher(ENVIRONMENTS.BASE_API_URL);

export const nextApi = new Fetcher(ENVIRONMENTS.NEXT_API_BASE_URL);



```
